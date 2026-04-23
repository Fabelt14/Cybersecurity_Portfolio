# Simulating Network Routing and VLAN Configuration in Linux

## Overview

This lab simulated enterprise network segmentation and routing using Linux network namespaces and kernel routing capabilities. Instead of requiring physical switches and routers, Linux's built-in networking features created isolated broadcast domains (VLANs) and controlled traffic flow between them using static routes and firewall rules. The goal was to understand how networks enforce security boundaries, route packets between subnets, and troubleshoot connectivity failures by analyzing traffic at the packet level.

## Objectives

- Configure static routing in the Linux kernel's Forwarding Information Base (FIB)
- Create isolated network segments using network namespaces to simulate VLANs
- Implement IP subnetting with proper boundary calculation and mask assignment
- Test inter-VLAN routing and diagnose connectivity issues using ping and traceroute
- Apply iptables firewall rules to control traffic forwarding between network segments
- Monitor packet flow using tcpdump to verify routing behavior and detect misconfigurations

## Lab Environment

- **Platform:** Kali Linux (192.168.92.4) with two network interfaces (eth0, eth1)
- **Target:** OWASP Broken Web Applications VM (192.168.92.3)
- **Network Topology:** Two simulated VLANs (vlan1: 192.168.0.0/25, vlan2: 192.168.0.128/25) connected through the default namespace acting as a router
- **Routing Method:** Static routes manually configured in the kernel

## Tools Used

- ip route (kernel routing table manipulation)
- ip netns (network namespace creation and management)
- ip addr (IP address assignment with subnet masks)
- ip link (virtual Ethernet pair creation)
- iptables (packet filtering and forwarding rules)
- tcpdump (packet capture and protocol analysis)
- ping (ICMP connectivity testing)
- traceroute (route path discovery)

## Methodology

### Exercise 1: Static Routing Configuration

Static routing teaches packets where to go by manually populating the kernel's routing table. Unlike dynamic routing protocols that discover routes automatically, static routing requires explicit administrator input for every network destination.

I added a route to the 192.168.1.0/24 subnet via the OWASP VM at 192.168.92.3:

```bash
sudo ip route add 192.168.1.0/24 via 192.168.92.3 dev eth1
```



![Routing table showing static route to 192.168.1.0/24](screenshots/static_route_added.png)



**How static routing works at the kernel level:**

When an application sends a packet to an IP address, the kernel consults its Forwarding Information Base (FIB). The FIB contains a list of network prefixes and their associated next-hop gateways. The kernel performs longest prefix matching: if a packet is destined for 192.168.1.50, the kernel finds the most specific route that covers that address.

In this case, the route entry states: "For any destination in 192.168.1.0/24, send the packet to 192.168.92.3 via the eth1 interface." The kernel then uses ARP to resolve 192.168.92.3 to a MAC address and transmits the frame.

**Why the command structure matters:**

- **Destination network (192.168.1.0/24):** The subnet this route applies to
- **via 192.168.92.3:** The next-hop gateway that knows how to reach 192.168.1.0/24
- **dev eth1:** The physical interface to use for transmission

If I omit "dev eth1," the kernel will choose an interface automatically based on which interface has a route to the gateway. Explicitly specifying the device prevents ambiguity in multi-homed systems.

**Challenges with manual static routing:**

**Scalability failure:** Large networks have hundreds or thousands of subnets. Manually adding routes for each one is impossible. A single typo in network prefix or gateway IP breaks connectivity silently. Dynamic routing protocols (OSPF, BGP) exchange route information automatically, eliminating manual configuration.

**No automatic failover:** If the gateway (192.168.92.3) becomes unreachable due to hardware failure or network outage, the static route remains in the routing table. The kernel continues trying to send packets to a dead gateway instead of finding an alternate path. Dynamic routing protocols detect failures and converge to working routes within seconds.

**Asymmetric routing risk:** I configured a route from Kali to 192.168.1.0/24 via the OWASP VM. But if the OWASP VM doesn't have a return route back to Kali's subnet, packets will arrive but responses will fail. Ping shows "Destination Host Unreachable" even though the outbound path works. Both sides must have complementary routes for bidirectional communication.

### Exercise 2: VLAN Simulation Using Network Namespaces

VLANs logically segment a network even when devices are physically connected to the same switch. Traditional VLANs use 802.1Q tagging to mark frames with a VLAN ID. Linux network namespaces achieve the same isolation without requiring physical VLAN-capable switches.

**How network namespaces simulate VLANs:**

Each namespace gets its own isolated network stack: separate routing table, ARP cache, firewall rules, and interface list. Processes running inside a namespace see only the interfaces assigned to that namespace. Even though all namespaces share the same physical hardware, they cannot directly communicate.

**Creating two isolated VLANs:**

```bash
sudo ip netns add vlan1
sudo ip netns add vlan2
```

These commands create two completely isolated network environments. If I run `ip addr` inside vlan1, it won't show any of the interfaces that exist in vlan2 or the default namespace.

**Benefits of namespace-based isolation:**

**Security and segmentation:** If a web server running in vlan1 is compromised, the attacker cannot pivot to database servers running in vlan2. Each namespace has separate firewall rules and routing tables. This implements Zero Trust Architecture where no network segment implicitly trusts another segment.

In a real environment, this could separate:
- Guest Wi-Fi (untrusted) from corporate LAN (trusted)
- Payment processing systems (PCI DSS compliance zone) from general business systems
- Development environment from production environment

**Resource overlap:** Two different applications can bind to the same port number (e.g., port 80) as long as they're in different namespaces with different IP addresses. This is impossible in a single namespace because port numbers must be unique per IP address.

Example: Running two web servers on the same physical machine:
- vlan1: 192.168.0.1:80 (public website)
- vlan2: 192.168.0.129:80 (internal admin panel)

Without namespaces, the second server would fail with "Address already in use" because port 80 is taken. Namespaces allow port reuse.



![Ping from vlan1 namespace to OWASP VM showing successful connectivity](screenshots/vlan1_ping.png)

Testing connectivity from inside the vlan1 namespace to the OWASP VM confirms that the namespace has a working network stack. The ping succeeds with 0% packet loss, proving the namespace can reach external destinations through the default namespace acting as a router.

### Exercise 3: IP Address Assignment and Subnetting

Subnetting divides a large network into smaller segments. The subnet mask determines which portion of the IP address represents the network and which portion represents individual hosts.

**Creating virtual Ethernet pairs to connect namespaces:**

```bash
sudo ip link add veth1 type veth peer name veth1-br
sudo ip link add veth2 type veth peer name veth2-br
```

These commands create two pairs of virtual network cables. One end (veth1) goes into the vlan1 namespace. The other end (veth1-br) stays in the default namespace. Packets sent to veth1 emerge from veth1-br, and vice versa. This simulates a physical cable connecting a switch port to a VLAN-tagged interface.

**Assigning IP addresses with /25 subnet masks:**

```bash
sudo ip netns exec vlan1 ip addr add 192.168.0.1/25 dev veth1
sudo ip netns exec vlan2 ip addr add 192.168.0.129/25 dev veth2
```



![IP address assignment showing vlan1 and vlan2 with /25 masks](screenshots/vlan_ip_assignment.png)



**How /25 subnetting works:**

A /25 mask (255.255.255.128) uses 25 bits for the network portion and 7 bits for hosts:

- **Network portion:** 192.168.0.NNNNNNN (first 25 bits)
- **Host portion:** .HHHHHHH (last 7 bits)

For 192.168.0.0/25:
- Network address: 192.168.0.0 (all host bits zero)
- First usable host: 192.168.0.1
- Last usable host: 192.168.0.126
- Broadcast address: 192.168.0.127 (all host bits one)

For 192.168.0.128/25:
- Network address: 192.168.0.128
- First usable host: 192.168.0.129
- Last usable host: 192.168.0.254
- Broadcast address: 192.168.0.255

These are two completely separate subnets. Even though they're in the same /24 block numerically, the /25 mask divides them into distinct broadcast domains.

**Configuring inter-VLAN routing:**

For vlan1 to reach vlan2, both namespaces need routes pointing to the default namespace as the gateway:

```bash
sudo ip netns exec vlan1 ip route add 192.168.0.128/25 dev veth1
sudo ip netns exec vlan2 ip route add 192.168.0.0/25 dev veth2
```

These routes say: "To reach the other subnet, send packets to the local veth interface. The other end of that virtual cable is in the default namespace, which will forward the packet to the destination namespace."



![Ping from vlan1 to vlan2 showing successful inter-VLAN routing](screenshots/vlan1_to_vlan2_ping.png)



The successful ping from 192.168.0.1 to 192.168.0.129 proves that routing between namespaces works. Packets travel: vlan1 → veth1 → veth1-br (default namespace) → routing decision → veth2-br → veth2 → vlan2.

**Challenges with manual subnetting:**

**Boundary errors:** Using 192.168.0.128 as a host IP in the first subnet fails because .128 is the network address of the second subnet. Similarly, using .127 as a host IP fails because it's the broadcast address of the first subnet. Linux silently rejects these assignments or routing fails mysteriously.

**Mask mismatches:** If vlan1 is configured with /25 (255.255.255.128) but vlan2 is mistakenly configured with /24 (255.255.255.0), routing becomes asymmetric:

- vlan1 thinks vlan2 is on a different network → sends packets through the router
- vlan2 thinks vlan1 is on the same /24 network → tries to ARP directly

Packets from vlan1 to vlan2 work (routed). Replies from vlan2 to vlan1 fail (ARP request goes unanswered because vlan1 isn't on the same segment). The result is one-way communication that's difficult to diagnose without packet captures.

### Exercise 4: Connectivity Testing and Route Discovery

Ping confirms that packets reach the destination. Traceroute reveals the path packets take to get there.



![Ping showing 0% packet loss to OWASP VM](screenshots/ping_success.png)



The ping results show:
- 5 packets transmitted, 5 received, 0% packet loss
- Average round-trip time: 0.407 ms

This extremely low latency (sub-millisecond) indicates the destination is on the local network with no intervening routers. The TTL of 64 in the ICMP replies confirms the OWASP VM is running Linux (Linux defaults to TTL=64).



![Traceroute showing single hop to destination](screenshots/traceroute_single_hop.png)



Traceroute output:
```
1  192.168.92.3 (192.168.92.3)  0.477 ms  0.384 ms  0.212 ms
```

**What traceroute reveals:**

The destination is reached in a single hop (no intermediate routers). The three time measurements (0.477 ms, 0.384 ms, 0.212 ms) represent three separate probe packets. The variation (0.477 → 0.212) is minimal, indicating stable routing with no congestion.

**How traceroute detects routing problems:**

**Routing loops:** If traceroute shows the same router appearing twice in the path, packets are circling between two routers that both think the other one is the next hop. This is caused by conflicting routing table entries.

**Inefficient routes:** If packets to a destination in the same city traverse three routers in different countries before arriving, routing is suboptimal. This happens when BGP prefers certain paths based on policy rather than geographic proximity.

**Blackhole detection:** If traceroute stops at hop 5 and never reaches the destination, hop 5 is the failure point. Either hop 5 is dropping packets, or the route beyond hop 5 doesn't exist.

**How routing affects performance:**

**Latency from hop count:** Each router adds processing delay (typically 1-10 ms for modern routers). A 10-hop path to a server has baseline latency 10x higher than a 1-hop path, even if bandwidth is identical.

**Throughput limited by slowest link:** If the path includes a 100 Mbps link, the entire connection is limited to 100 Mbps even if all other links are 10 Gbps. The slowest link creates a bottleneck.

**Jitter from route instability:** If packets alternate between a 5-hop path and a 10-hop path due to routing protocol convergence, latency varies unpredictably. This creates jitter that degrades VoIP call quality and video streaming.

**CPU overhead on routers:** Complex routing decisions (longest prefix matching in routing tables with 800,000+ entries) consume CPU cycles. Under high traffic load, routers can become CPU-bound, causing packet drops even when bandwidth is available.

### Exercise 5: Firewall Rules and Traffic Forwarding

The Linux kernel can forward packets between interfaces, effectively turning a server into a router. But forwarding is disabled by default for security.

**Enabling IP forwarding:**

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```



![IP forwarding enabled in kernel](screenshots/ip_forward_enabled.png)



This tells the kernel: "When a packet arrives on one interface with a destination IP that belongs to a different interface, forward it instead of dropping it." Without this, the kernel only processes packets destined for its own IP addresses.

**Configuring iptables forwarding rules:**

```bash
sudo iptables -A FORWARD -s 192.168.0.0/25 -d 192.168.0.128/25 -j ACCEPT
sudo iptables -A FORWARD -s 192.168.0.0/25 -d 192.168.92.3 -j DROP
```



![iptables FORWARD rules controlling inter-VLAN traffic](screenshots/iptables_forward_rules.png)



**How iptables FORWARD chain works:**

The routing table decides where packets should go. The FORWARD chain decides whether they're allowed to go there. Even if a route exists, packets are dropped if the FORWARD chain rejects them.

Rule 1: "Allow packets from vlan1 (192.168.0.0/25) to vlan2 (192.168.0.128/25)." This permits inter-VLAN routing.

Rule 2: "Drop packets from vlan1 (192.168.0.0/25) to the OWASP VM (192.168.92.3)." This blocks vlan1 from accessing external resources while still allowing vlan1-to-vlan2 communication.

**Why order matters:**

iptables processes rules top to bottom. The first matching rule determines the packet's fate. If rule 2 appeared before rule 1 and was written as "DROP all packets from 192.168.0.0/25," rule 1 would never be reached. All traffic from vlan1 would be blocked, including legitimate vlan1-to-vlan2 traffic.

**NAT table for address translation:**

The NAT (Network Address Translation) table allows private namespaces to share a single public IP:

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE
```

This rule says: "For packets from the 192.168.0.0/24 network leaving through eth0 (internet-facing interface), replace the source IP with eth0's IP address."

When vlan1 (192.168.0.1) sends a packet to an internet server, the NAT table rewrites:
- Original: source=192.168.0.1, destination=8.8.8.8
- After NAT: source=10.0.2.15 (eth0's IP), destination=8.8.8.8

The internet server replies to 10.0.2.15. The NAT table maintains a connection tracking table that remembers "10.0.2.15:5000 maps to 192.168.0.1:5000" and translates the response back.

**Common iptables mistakes:**

**Permit-all rule at the top:** If the first rule is `-A FORWARD -j ACCEPT`, every packet is allowed. All subsequent deny rules are ignored. The default-deny approach (block everything, then explicitly permit) is safer.

**Rules lost on reboot:** iptables rules are stored in RAM. Rebooting clears all rules. Production systems use `iptables-save > /etc/iptables/rules.v4` to persist rules, then load them at boot with `iptables-restore`.

**Missing state tracking:** Without connection tracking (`-m state --state ESTABLISHED,RELATED`), return traffic is blocked. The initial packet is allowed, but the response is dropped because it doesn't match any rule.

### Exercise 6: Traffic Monitoring with tcpdump

tcpdump captures raw packets, revealing what's actually happening on the wire versus what should be happening according to configuration.

**Capturing traffic from vlan1 namespace:**

```bash
sudo ip netns exec vlan1 tcpdump -i veth1
```



![tcpdump showing ICMP echo requests and replies between VLANs](screenshots/tcpdump_icmp_vlan.png)

The capture shows ICMP echo request and reply packets:

```
15:15:21.802567 IP 192.168.0.1 > 192.168.0.129: ICMP echo request, id 6028, seq 1
15:15:21.802588 IP 192.168.0.129 > 192.168.0.1: ICMP echo reply, id 6028, seq 1
```

This confirms bidirectional routing works. The request travels from vlan1 to vlan2, and the reply successfully returns. If the reply was missing, it would indicate asymmetric routing (outbound route exists, return route doesn't).

**Capturing traffic on the main interface:**



![tcpdump showing HTTP, ARP, and TCP traffic](screenshots/tcpdump_mixed_traffic.png)



The capture shows mixed protocol traffic:

- **HTTP:** Web requests to the OWASP VM
- **ARP:** Address resolution requests/replies mapping IP addresses to MAC addresses
- **TCP:** Three-way handshakes (SYN, SYN-ACK, ACK) establishing connections

**How tcpdump diagnoses failures:**

**Firewall drops:** If tcpdump on the source shows packets leaving but tcpdump on the destination shows no packets arriving, a firewall between them is dropping traffic. The firewall doesn't send rejection messages, packets just vanish.

**Failed TCP handshakes:** If tcpdump shows SYN packets sent but no SYN-ACK responses, either:
- The destination isn't listening on that port (would send RST)
- A firewall is blocking the SYN packets (no response at all)
- Routing is broken and SYN packets never arrive

**Configuration errors:** If tcpdump shows packets with the wrong source IP or missing VLAN tags, the interface configuration is wrong. The application thinks it's sending from one IP, but the kernel is using a different source IP due to routing policy.

**Subnet mask mismatches:** If tcpdump shows ARP requests for an IP that should be routed, the subnet mask is wrong. The sender thinks the destination is local when it's actually remote, so it ARPs instead of routing through the gateway.

## Findings

**Static routing requires perfect configuration on both ends.** A route from A to B is useless if B doesn't have a return route to A. Both sides must have complementary routing table entries for bidirectional communication. Missing or incorrect routes cause one-way connectivity that appears intermittent and is difficult to troubleshoot without packet captures.

**Network namespaces provide complete isolation without physical hardware.** Each namespace has its own routing table, ARP cache, firewall rules, and interface list. Processes in one namespace cannot see or interact with processes in another namespace. This simulates physical VLANs using only software, enabling security segmentation on commodity hardware.

**Subnet masks define network boundaries at the bit level.** A /25 mask splits a /24 network in half. The first subnet uses .0-.127, the second uses .128-.255. Using .0 or .128 as a host IP fails because those are network addresses. Using .127 or .255 fails because those are broadcast addresses. Subnetting requires precise boundary calculation.

**Routing tables decide destination, firewalls decide permission.** Even with a route to the destination in the routing table, iptables FORWARD rules can block the packet. The routing table answers "where should this go?" The firewall answers "is it allowed to go there?" Both must permit the traffic for communication to succeed.

**Packet captures show ground truth.** Applications might report "connection failed" without explaining why. tcpdump shows whether packets are being sent, whether they're arriving, what responses are received, and where communication breaks down. This eliminates guesswork in troubleshooting.

**IP forwarding must be explicitly enabled.** By default, Linux doesn't route packets between interfaces. This prevents servers from accidentally becoming routers and forwarding attack traffic. Enabling forwarding converts a server into a router, requiring firewall rules to prevent abuse.

**iptables rule order determines behavior.** Rules are processed top to bottom. The first match wins. A permit-all rule at the top makes all subsequent deny rules unreachable. Production firewalls use default-deny (drop everything at the bottom, explicitly permit above) to prevent configuration errors from creating security holes.

## Challenges Faced

**Asymmetric routing confusion:** Initially, pings from vlan1 to vlan2 succeeded, but pings from vlan2 to vlan1 failed. I configured routes in vlan1 but forgot to add the complementary route in vlan2. The kernel in vlan2 thought vlan1 was unreachable and dropped reply packets. tcpdump showed ICMP echo requests arriving at vlan1 but no replies leaving. This taught me that routes must exist on both sides.

**Subnet boundary calculation errors:** I initially tried to use 192.168.0.128 as a host IP in the first /25 subnet. The assignment succeeded, but routing failed silently. Only after binary conversion did I realize .128 is the network address of the second subnet, not a valid host in the first subnet. Subnetting requires understanding bit-level boundaries, not just decimal arithmetic.

**iptables rule testing without persistence:** After configuring complex firewall rules, I rebooted to test another exercise. All iptables rules vanished because I didn't run `iptables-save`. I had to recreate all rules from memory. This taught me that firewall configuration is volatile by default and must be explicitly persisted to survive reboots.

**Namespace interface visibility confusion:** Running `ip addr` showed interfaces that I thought were in a namespace but were actually in the default namespace. Each namespace has completely separate interface lists. To see interfaces inside a namespace, I must run `sudo ip netns exec vlan1 ip addr`, not just `ip addr`. Without the `ip netns exec` prefix, I was viewing the default namespace.

## Key Takeaways

**Static routing doesn't scale, but it teaches routing fundamentals.** Manually adding routes for every subnet is impractical in production networks. Dynamic routing protocols (OSPF, BGP) exchange routes automatically. But understanding how the kernel's routing table makes forwarding decisions is essential for troubleshooting, even in dynamically routed networks.

**Network segmentation is a security requirement, not just organizational convenience.** Isolating guest Wi-Fi from corporate LAN, payment systems from general business systems, and development from production prevents lateral movement after compromise. Namespaces demonstrate how isolation works at the kernel level.

**Subnetting is binary math, not decimal math.** A /25 mask is 11111111.11111111.11111111.10000000 in binary. The boundary falls at bit 25. Understanding this prevents off-by-one errors when calculating usable host ranges.

**Firewalls control traffic flow independent of routing.** A route in the routing table means the kernel knows how to deliver the packet. An ACCEPT rule in the FORWARD chain means the packet is permitted to be delivered. Both are required. Configuring routes without firewall rules creates security vulnerabilities. Configuring firewall rules without routes creates connectivity failures.

**Diagnostic tools answer specific questions.** Ping answers "can I reach this destination?" Traceroute answers "what path do packets take?" tcpdump answers "what packets are actually being sent and received?" Each tool provides different information. Using the right tool for the question eliminates guesswork.

**Namespaces enable multi-tenancy without virtualization.** Running multiple isolated network stacks on the same physical host enables container networking (Docker, Kubernetes), ISP customer isolation, and lab environments without requiring virtual machines. Understanding namespaces is fundamental to modern infrastructure.

## Disclaimer

This lab was performed in a controlled virtual environment using network namespaces and virtual Ethernet pairs. No production networks were modified. All routing configurations were temporary and discarded after the exercise. The techniques demonstrated (static routing, firewall rules, network segmentation) are standard network administration practices used in production environments, but this specific implementation was for educational purposes only.