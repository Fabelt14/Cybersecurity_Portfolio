# **Packet Tracer: Logical and Physical Mode Exploration**

## **Overview**

This lab introduces Packet Tracer's logical and physical network views to understand how network devices are interconnected. The goal was to navigate between different perspectives, identify device connections, and perform basic device configurations in a simulated campus network environment.

## **Objectives**

- Explore Packet Tracer's interface and network device categories
- Identify physical device connections in a wiring closet
- Connect end devices using appropriate cable types
- Install and configure a backup router
- Configure basic device hostnames via CLI
- Understand the relationship between logical and physical network topologies

## **Lab Environment**

- Platform: Cisco Packet Tracer
- Network Scenario: Intercity network with multiple switches, routers, and wireless access points
- Topology Type: Mixed (wired and wireless connections)
- Configuration Method: CLI (Command Line Interface)

## **Tools Used**

- Cisco Packet Tracer (network simulation tool)
- Various cable types:
  - Copper Straight-Through cable
  - Console cable (RS232)
  - USB console cable
 
## **Methodology**

### **Part 1: Investigate the Bottom Toolbar**
I started by exploring Packet Tracer's bottom toolbar to familiarize myself with available network device categories. This helps understand what infrastructure components exist in real networks.

**Devices identified:**

- Routers
- Switches
- Hubs
- Wireless Devices
- Security appliances
- WAN Emulation devices

![Network Devices](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/network%20devices.jpg)

### **Part 2: Investigate Devices in a Wiring Closet**
I investigated the physical connections in a wiring closet to understand how enterprise networks are physically structured.

**Question 1: What devices use wired connections to switch ALS2?**

**Findings:**
- Webserver
- Access Point
- ALS1 switch

![Wiring Closet](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Wiring%20closets.jpg)

**Question 2: Which device is connected to Access_Point?**

**Finding:**
- Laptop_1

![Laptop1](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Access%20points.jpg)


**Question 3: Where is the device connected to Access_Point physically located?**

**Answer:** It is located on the right side of the table

![physical location](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/physical%20location.jpg)


### **Part 3: Connect End Devices to Networking Devices**

I connected PC_1 to the ALS2 Switch using a copper straight-through cable.

![PC to Switch](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Cable%20to%20ALS2.jpg)

I also connected PC_1 to the Edge_Router on a console port (RS-232) using a console cable.

![PC to Edge Router](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Cable%20to%20Edge%20router.jpg)


### **Part 4: Install a Backup Router**

I connected Laptop_1 to Backup_Router on a USB port using a USB console cable.

![Backup Router](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/installing%20backup%20router.jpg)


### **Part 5: Configure a Hostname**

I configured the name of the Backup_Router to Edge_Router_Backup using CLI commands

**CLI Command used:**

> enable

> configure terminal

> hostname Edge_Router_Backup

> end

![hostame](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/config%20hostname.jpg)


### **Activity Completed**

![Lab Completed 1](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Lab%20result%201.jpg)


![Lab Completed II](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Lab%20result%202.jpg)

![Overall Network](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/overall%20networks.jpg)


## **Key Takeaways**

- Physical and logical views provide different but complementary perspectives on network architecture
- Using the wrong cables in real environments can cause connectivity issues or damage
- Device naming conventions reveal organizational structure and can aid in reconnaissance

## **Disclaimer**

This lab was performed in a Cisco Packet Tracer simulation environment for educational purposes only. No physical or production network devices were accessed.

