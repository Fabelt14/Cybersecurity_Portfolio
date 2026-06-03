Lab 01: Navigate Cisco IOS Using Console Connectivity

Overview

This lab focused on accessing Cisco networking devices through console connections and navigating the Cisco IOS Command-Line Interface (CLI). The objective was to establish out-of-band management access to both a Cisco 2960 switch and a Cisco 4321 router using console cables and a terminal client.

Learning Objectives

- Access a Cisco switch through a serial console connection
- Access a Cisco router through a Mini-USB console connection
- Navigate Cisco IOS command modes
- Display device information using IOS commands
- Configure basic device settings
- Understand out-of-band management concepts

---

Lab Topology

Switch Console Connection

+-----------+       Console Cable       +-------------+
|    PC     | ------------------------> | Cisco 2960  |
| (RS-232)  |                           |   Switch    |
+-----------+                           +-------------+

Router Console Connection

+-----------+       Mini USB Cable      +-------------+
|  Laptop   | ------------------------> | Cisco 4321  |
|           |                           |   Router    |
+-----------+                           +-------------+

---

Devices Used

Cisco Devices

- Cisco Catalyst 2960 Switch
- Cisco ISR 4321 Router

End Devices

- PC
- Laptop

Cables

- Rollover Console Cable
- Mini-USB Console Cable

---

Part 1: Accessing a Cisco Switch Through the Console Port

Device Inspection

After installing the Cisco 2960 switch, the following interfaces were identified:

Front Panel

- FastEthernet 0/1 - 0/24
- GigabitEthernet 0/1 - 0/2

Rear Panel

- Console Port

PC Inspection

The following interfaces were identified on the PC:

- FastEthernet Port
- RS-232 Serial Port
- USB Ports

Console Connection

A rollover console cable was used to connect:

PC RS-232 Port --> Cisco 2960 Console Port

Outcome

Successfully established console access to the switch.

---

Part 2: Displaying and Configuring Basic Device Settings

Display IOS Version

Command used:

show version

Observation

The switch was running Cisco IOS Version 12.2.

Configure Device Clock

The system clock was found to be incorrect.

Steps performed:

1. Entered Privileged EXEC mode
2. Configured the correct date and time

Outcome

Successfully updated the switch clock settings.

---

Part 3: Accessing a Router Through a Mini-USB Console Connection

Router Inspection

The following interfaces and components were identified:

- Power Button
- USB Console Port
- Auxiliary Port
- Console Port
- GigabitEthernet Interfaces

Laptop Inspection

The following interfaces were identified:

- RS-232 Port
- FastEthernet Port
- USB Ports

Console Connection

A Mini-USB cable was used to connect:

Laptop USB Port --> Cisco 4321 USB Console Port

Establish Terminal Session

After connecting the cable, a terminal session was launched.

Successful access was confirmed by the router prompt:

Router>

Outcome

Successfully established console access to the router.

---

Verification

Switch Verification

show version

Verified:

- IOS Version
- Device Information

Router Verification

Router>

Verified:

- Successful console connectivity
- Access to IOS CLI

---

Key Concepts Learned

Out-of-Band Management

Out-of-band management allows administrators to access network devices through a dedicated management connection instead of relying on network connectivity.

Cisco IOS Modes

- User EXEC Mode (">")
- Privileged EXEC Mode ("#")
- Configuration Modes

Console Access

Console connections provide direct access to networking devices for initial configuration, troubleshooting, and recovery.

---

Challenges Encountered

- Identifying the correct console ports on each device
- Understanding the difference between console access and normal network access
- Navigating between IOS command modes

---

Lessons Learned

1. Cisco devices can be managed without an active network connection using console access.
2. The "show version" command provides valuable information about a device's software and hardware.
3. Understanding IOS command modes is essential for network administration.
4. Console connectivity is often the first step when configuring a new Cisco device.

---

Skills Practiced

- Cisco IOS Navigation
- Console Connectivity
- Device Inspection
- Basic Switch Management
- Basic Router Management
- Network Device Administration
- Out-of-Band Access

---

Lab Status

✅ Successfully Completed