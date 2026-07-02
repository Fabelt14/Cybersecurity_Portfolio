# Navigate the IOS Using a Terminal Client for Console Connectivity - Physical Mode

## Objective

The goal of this lab was to establish console connectivity between end devices and Cisco network equipment using physical cable connections. This covered two connection types: a rollover console cable between a PC and a Cisco 2960 switch, and a mini-USB cable between a laptop and a Cisco 4321 router. Once connected, basic IOS navigation and device configuration were performed through the CLI.

---

## Topology

**Part 1: PC to Cisco 2960 Switch (Rollover Console Cable)**

![PC connected to 2960 Switch via RS-232 rollover console cable](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_10%20PC%20connected%20to%202960%20Switch%20via%20RS-232%20rollover%20console%20cable.jpg)

*Source Device: PC | Source Port: RS 232 | Dest. Device: 2960 | Dest. Port: Console | Cable Type: Copper Roll-Over*

**Part 3: Laptop to Cisco 4321 Router (Mini-USB Cable)**

![Laptop connected to 4321 Router via USB Console cable](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_11%20Laptop%20connected%20to%204321%20Router%20via%20USB%20Console%20cable.jpg)

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

### Part 1: Cisco 2960 Switch: Physical Inspection

**Front view of the 2960 Switch:**

![Front view of Cisco 2960 Switch showing FastEthernet and GigabitEthernet ports](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_01%20Front%20view%20of%20Cisco%202960%20Switch%20showing%20FastEthernet%20and%20GigabitEthernet%20ports.jpg)

Ports identified: FastEthernet 0/1 – 0/24 and GigabitEthernet 0/1 – 0/2.

**Rear view of the 2960 Switch:**

![Rear view of Cisco 2960 Switch showing the Console Port](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_02%20Rear%20view%20of%20Cisco%202960%20Switch%20showing%20the%20Console%20Port.jpg)

The console port used for out-of-band management access was identified on the rear panel.


**Front view of the PC:**

![Front view of PC showing RS-232 port, FastEthernet port, and 2 USB Console ports](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_03%20Front%20view%20of%20PC%20showing%20RS-232%20port%2C%20FastEthernet%20port%2C%20and%202%20USB%20Console%20ports.jpg)

Ports identified: FastEthernet port, RS-232 port, and 2 USB ports for console access.

---

### Part 2: IOS Version: show version Output

After establishing the console session, the `show version` command was run to confirm the IOS version running on the switch.

![show version output confirming Cisco IOS Version 12.2 on the 2960 switch](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_04%20show%20version%20output%20confirming%20Cisco%20IOS%20Version%2012.2%20on%20the%202960%20switch.jpg)

IOS version confirmed: **12.2** (C2960-LANBASE-M image, WS-C2960-24TT model).


The switch clock was set to the correct date and time after observing that the default clock was incorrect (showing March 1, 1993).

```
Switch> show clock
*0:15:20.898 UTC Mon Mar 1 1993

Switch> enable
Switch# clock set 01:10:55 19 May 2026
Switch# show clock
1:11:0.731 UTC Tue May 19 2026
```

![Clock configuration output on the 2960 switch showing corrected date and time](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_05%20Clock%20configuration%20output%20on%20the%202960%20switch%20showing%20corrected%20date%20and%20time.jpg)

---

### Part 3: Cisco 4321 Router: Physical Inspection

**Front view of the 4321 Router:**

![Front view of Cisco 4321 Router showing power button, USB console port, AUX port, Console port, and 3 GigabitEthernet ports](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_06%20Front%20view%20of%20Cisco%204321%20Router%20showing%20power%20button%2C%20USB%20console%20port%2C%20AUX%20port%2C%20Console%20port%2C%20and%203%20GigabitEthernet%20ports.jpg)

Components identified: power button, USB console port, auxiliary port, console port, and 3 GigabitEthernet ports (GE0/0/0, GE0/0/1, GE0/0/2).



**Rear view of the Laptop:**

![Rear view of Laptop showing RS-232 port, FastEthernet port, and 2 USB ports](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_07%20Rear%20view%20of%20Laptop%20showing%20RS-232%20port%2C%20FastEthernet%20port%2C%20and%202%20USB%20ports.jpg)

Ports identified: RS-232 port, FastEthernet port, and 2 USB ports.


After connecting the laptop to the 4321 router via mini-USB, the terminal was opened. The `Router>` prompt confirmed a successful console session.

![Terminal output showing Router> prompt after successful console connection via mini-USB](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_08%20Terminal%20output%20showing%20Router%20prompt%20after%20successful%20console%20connection%20via%20mini-USB.jpg)

---

## Verification

**Lab Score: 10/10**

![Packet Tracer Activity Results showing 10/10 score with all assessment items marked Correct](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/03_09%20Packet%20Tracer%20Activity%20Results%20showing%2010%20score%20with%20all%20assessment%20items%20marked%20Correct.jpg)

All assessment items passed under the Physical component category, covering correct cable connections, physical locations, and power states for the 2960 switch, 4321 router, laptop, and PC.

---

## Key Concepts Learned

- A rollover console cable connects a PC's RS-232 port to a switch's console port for out-of-band management
- A mini-USB cable connects a laptop's USB port to a router's USB console port for the same purpose
- Out-of-band access through the console port does not depend on the network being operational, this matters during initial device setup or recovery
- The `show version` command returns IOS version, model number, memory, and uptime, all relevant for assessing the device's current state
- The `show clock` command exposes incorrect time settings; accurate timestamps matter for log correlation during incident response
- Privilege EXEC mode (`enable`) is required before making any configuration changes

---

## Challenges Faced

No major blockers were encountered during this lab. The default clock on the switch showed an incorrect date (March 1, 1993), which was corrected by switching to Privilege EXEC mode and using the `clock set` command with the accurate date and time.

---

## Lessons Learned

- Console access is the foundation of device management. Before any network configuration happens, an administrator must be able to reach the device directly and that happens through the console port. Understanding which cable type matches which port (rollover vs. mini-USB) prevents wasted time during real deployments or emergency recovery.

- Seeing an incorrect system clock on a live switch would be a problem. Logs use timestamps. If the clock is wrong, log entries during an incident become unreliable, and correlating events across devices becomes harder. Correcting the clock is not a minor housekeeping task, it has a direct impact on forensic accuracy.

---

## Disclaimer

This lab was performed in a controlled Cisco Packet Tracer environment for educational purposes only. No unauthorized systems were accessed or tested.
