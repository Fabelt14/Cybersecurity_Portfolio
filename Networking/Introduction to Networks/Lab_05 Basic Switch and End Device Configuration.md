# Basic Switch and End Device Configuration

## Objective

Configure two Cisco switches with secure initial settings using the Cisco IOS, then assign IP addressing to two end devices so they can reach each other and the switches across a small LAN. This lab simulates the kind of baseline hardening a LAN technician applies before a switch goes into production: locking down access lines, encrypting credentials, and setting a banner that puts unauthorized users on notice before they get anywhere near a prompt.

---

## Topology

![Network topology showing Class-A switch connected to Student-1 PC and Class-B switch, which connects to Student-2 PC](images/basic-switch-and-end-device-configuration/network-topology.png)

Two switches, Class-A and Class-B, are connected to each other. Student-1 connects to Class-A and Student-2 connects to Class-B. Console cables run from each PC to its switch for initial configuration access.

---

## Devices Used

- Class-A switch
- Class-B switch
- Student-1 PC
- Student-2 PC

---

## Configuration

### Console Access

Access to both switches started through a console connection rather than Telnet or SSH, since neither switch had any remote management configured yet. Console access is the only option on a switch fresh out of the box, and it's also the safest way to make initial changes since it requires physical access to the device.

![Console connection established to Class-A and Class-B switches](images/basic-switch-and-end-device-configuration/console-connection.png)

### Hostname Configuration

Each switch was renamed from the default `Switch` to something that identifies its role on the network: Class-A and Class-B.

```
Switch(config)#hostname Class-A
```

Default hostnames are one of the first things an attacker checks for during a network walk-through. A device still showing `Switch>` or `Router>` is an easy signal that nobody has touched its configuration since it left the factory. Naming devices isn't just organizational, it removes that signal.

![Hostname changed from Switch to Class-A and Class-B](images/basic-switch-and-end-device-configuration/hostname-configuration.png)

### Line Security

Both the console line and all 16 virtual terminal (vty) lines on each switch were configured with the password `xAw6k` and login enabled.

```
Class-A(config)#line vty 0 15
Class-A(config-line)#password xAw6k
Class-A(config-line)#login
Class-A(config)#line console 0
Class-A(config-line)#password xAw6k
Class-A(config-line)#login
```

Leaving vty lines without a password is a common misconfiguration that turns Telnet or SSH into an open door. Setting `login` without a password locks an admin out, and setting a password without `login` doesn't actually enforce it, so both commands had to be present together to work as intended.

![Console and vty lines configured with password on Class-A and Class-B](images/basic-switch-and-end-device-configuration/line-password-configuration.png)

### Enable Secret

The privileged EXEC mode on both switches was protected with the secret `6EBUp`.

```
Class-A(config)#enable secret 6EBUp
```

The enable secret is hashed by default in the running configuration, unlike the plain `enable password` command, which stores the value in clear text. Using `secret` instead of `password` for privileged access is a small choice with a real difference in how exposed that credential is if someone gets a look at the config file.

![Enable secret configured on Class-A and Class-B](images/basic-switch-and-end-device-configuration/enable-secret-configuration.png)

### Password Encryption

```
Class-A(config)#service password-encryption
```

This command encrypts the line passwords (console and vty) that would otherwise sit in plain text in the configuration file. It's a weak, reversible encryption by modern standards, but it still beats leaving credentials fully readable to anyone who can view the config, whether that's over the shoulder, in a backup file, or in a config pasted into a support ticket.

![Service password-encryption applied to Class-A and Class-B](images/basic-switch-and-end-device-configuration/password-encryption.png)

### MOTD Banner

```
Class-A(config)#banner motd #Authorized Access Only!!!#
```

A message-of-the-day banner displays before login. Its real value is legal, not technical: it puts anyone connecting on notice that access is restricted, which matters if an organization ever needs to pursue action against unauthorized access. It's a small step that gets skipped more often than it should.

![MOTD banner displayed on Class-A and Class-B before login](images/basic-switch-and-end-device-configuration/motd-banner.png)

### IP Addressing

Each switch was given a management IP address on VLAN 1 so it could be reached for remote administration.

```
Class-A(config)#interface vlan 1
Class-A(config-if)#ip address 128.107.20.10 255.255.255.0
Class-A(config-if)#no shutdown
```

| Device | Interface | IP Address | Subnet Mask |
|---|---|---|---|
| Class-A | VLAN 1 | 128.107.20.10 | 255.255.255.0 |
| Class-B | VLAN 1 | 128.107.20.15 | 255.255.255.0 |
| Student-1 | NIC | *[fill in: IP not visible in lab screenshots]* | 255.255.255.0 |
| Student-2 | NIC | 128.107.20.30 | 255.255.255.0 |

![IP address and subnet mask assigned to Class-A and Class-B VLAN 1 interfaces](images/basic-switch-and-end-device-configuration/ip-addressing-configuration.png)

### Saving the Configuration

```
Class-A#copy running-config startup-config
```

Running configuration only lives in memory. If a switch loses power or reboots before this command runs, every change made in the session is gone. Saving to startup-config is the step that actually makes the work persistent.

![Running configuration saved to startup configuration on Class-A and Class-B](images/basic-switch-and-end-device-configuration/save-configuration.png)

---

## Verification

Connectivity was tested with ICMP pings between every device pair in the topology.

| Source | Destination | Result |
|---|---|---|
| Student-1 | Student-2 (128.107.20.30) | Successful |
| Class-A | Class-B (128.107.20.15) | Successful |
| Student-1 | Class-B (128.107.20.15) | Successful |
| Student-2 | Class-A (128.107.20.10) | Successful |
| Student-1 | Class-A (128.107.20.10) | Successful |
| Student-2 | Class-B (128.107.20.15) | Successful |

All six tests passed, confirming end-to-end reachability between both PCs and both switches.

![Connectivity test results showing all ICMP tests successful](images/basic-switch-and-end-device-configuration/connectivity-verification.png)

![Final assessment results showing completed activity score](images/basic-switch-and-end-device-configuration/assessment-results.png)

---

## Key Concepts Learned

- Console access is the baseline method for configuring a switch that has no prior management setup, and it's the most secure way to make first-touch changes since it requires physical presence.
- `enable secret` and `service password-encryption` exist because the default behavior of IOS stores credentials in plain text, which is a problem worth fixing before a device goes anywhere near production.
- A banner is a legal control, not a technical one. It doesn't stop an attacker, but it removes any ambiguity about whether access was authorized.
- VLAN 1 interface addressing gives a switch a management IP, which is separate from the data-plane switching it does by default at Layer 2.


---

## Disclaimer

This lab was performed in a controlled Cisco Packet Tracer environment for educational purposes only. No real or production network devices were configured or tested.