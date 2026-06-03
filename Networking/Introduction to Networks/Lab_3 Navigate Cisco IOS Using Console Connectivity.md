# Navigate the IOS Using a Terminal Client for Console Connectivity - Physical Mode

## Objective

The goal of this lab was to establish console connectivity between end devices and Cisco network equipment using physical cable connections. This covered two connection types: a rollover console cable between a PC and a Cisco 2960 switch, and a mini-USB cable between a laptop and a Cisco 4321 router. Once connected, basic IOS navigation and device configuration were performed through the CLI.

---

## Topology

**Part 1 — PC to Cisco 2960 Switch (Rollover Console Cable)**

![PC connected to 2960 Switch via RS-232 rollover console cable](images/topology-pc-switch-rollover.png)

*Source Device: PC | Source Port: RS 232 | Dest. Device: 2960 | Dest. Port: Console | Cable Type: Copper Roll-Over*

**Part 3 — Laptop to Cisco 4321 Router (Mini-USB Cable)**

![Laptop connected to 4321 Router via USB Console cable](images/topology-laptop-router-usb.png)

*Source Device: Laptop | Source Port: USB0 | Dest. Device: 4321 | Dest. Port: USB Console | Cable Type: USB*

---

## Devices Used

- Cisco Catalyst 2960-24TT Switch
- Cisco 4321 Router
- PC (with RS-232 and USB ports)
- Laptop (with RS-232 and USB ports)
- Rollover Console Cable (Copper Roll-Over)
- Mini-USB Console Cable

---

## Configuration

### Part 1: Cisco 2960 Switch — Physical Inspection

**Front view of the 2960 Switch:**

![Front view of Cisco 2960 Switch showing FastEthernet and GigabitEthernet ports](images/switch-2960-front.png)

Ports identified: FastEthernet 0/1 – 0/24 and GigabitEthernet 0/1 – 0/2.

**Rear view of the 2960 Switch:**

![Rear view of Cisco 2960 Switch showing the Console Port](images/switch-2960-rear.png)

The console port used for out-of-band management access was identified on the rear panel.

---

### Part 1: PC — Physical Inspection

**Front view of the PC:**

![Front view of PC showing RS-232 port, FastEthernet port, and 2 USB Console ports](images/pc-front.png)

Ports identified: FastEthernet port, RS-232 port, and 2 USB ports for console access.

---

### Part 2: IOS Version — show version Output

After establishing the console session, the `show version` command was run to confirm the IOS version running on the switch.

![show version output confirming Cisco IOS Version 12.2 on the 2960 switch](images/show-version-output.png)

IOS version confirmed: **12.2** (C2960-LANBASE-M image, WS-C2960-24TT model).

---

### Part 2: Clock Configuration

The switch clock was set to the correct date and time after observing that the default clock was incorrect (showing March 1, 1993).

```
Switch> show clock
*0:15:20.898 UTC Mon Mar 1 1993

Switch> enable
Switch# clock set 01:10:55 19 May 2026
Switch# show clock
1:11:0.731 UTC Tue May 19 2026
```

![Clock configuration output on the 2960 switch showing corrected date and time](images/clock-config-output.png)

---

### Part 3: Cisco 4321 Router — Physical Inspection

**Front view of the 4321 Router:**

![Front view of Cisco 4321 Router showing power button, USB console port, AUX port, Console port, and 3 GigabitEthernet ports](images/router-4321-front.png)

Components identified: power button, USB console port, auxiliary port, console port, and 3 GigabitEthernet ports (GE0/0/0, GE0/0/1, GE0/0/2).

---

### Part 3: Laptop — Physical Inspection

**Rear view of the Laptop:**

![Rear view of Laptop showing RS-232 port, FastEthernet port, and 2 USB ports](images/laptop-rear.png)

Ports identified: RS-232 port, FastEthernet port, and 2 USB ports.

---

### Part 3: Router Console Session

After connecting the laptop to the 4321 router via mini-USB, the terminal was opened. The `Router>` prompt confirmed a successful console session.

![Terminal output showing Router> prompt after successful console connection via mini-USB](images/router-console-session.png)

---

## Verification

**Lab Score: 10/10**

![Packet Tracer Activity Results showing 10/10 score with all assessment items marked Correct](images/activity-results-score.png)

All assessment items passed under the Physical component category, covering correct cable connections, physical locations, and power states for the 2960 switch, 4321 router, laptop, and PC.

---

## Key Concepts Learned

- A rollover console cable connects a PC's RS-232 port to a switch's console port for out-of-band management
- A mini-USB cable connects a laptop's USB port to a router's USB console port for the same purpose
- Out-of-band access through the console port does not depend on the network being operational — this matters during initial device setup or recovery
- The `show version` command returns IOS version, model number, memory, and uptime — all relevant for assessing the device's current state
- The `show clock` command exposes incorrect time settings; accurate timestamps matter for log correlation during incident response
- Privilege EXEC mode (`enable`) is required before making any configuration changes

---

## Challenges Faced

No major blockers were encountered during this lab. The default clock on the switch showed an incorrect date (March 1, 1993), which was corrected by switching to Privilege EXEC mode and using the `clock set` command with the accurate date and time.

---

## Lessons Learned

Console access is the foundation of device management. Before any network configuration happens, an administrator must be able to reach the device directly — and that happens through the console port. Understanding which cable type matches which port (rollover vs. mini-USB) prevents wasted time during real deployments or emergency recovery.

Seeing an incorrect system clock on a live switch would be a problem. Logs use timestamps. If the clock is wrong, log entries during an incident become unreliable, and correlating events across devices becomes harder. Correcting the clock is not a minor housekeeping task — it has a direct impact on forensic accuracy.

---

## Disclaimer

This lab was performed in a controlled Cisco Packet Tracer environment for educational purposes only. No unauthorized systems were accessed or tested.
