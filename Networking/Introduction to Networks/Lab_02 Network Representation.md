# Packet Tracer - Network Representation

## Overview

This lab focused on identifying and understanding network components using Cisco Packet Tracer. The goal was to recognize different device types, understand their roles, and distinguish between LANs and WANs in a simulated small-to-medium business network.

## Objectives

- Identify intermediary devices, end devices, and media types in a network topology
- Explain the purpose and function of different network devices
- Understand the client-server model
- Differentiate between LAN and WAN connections
- Analyze a business network structure

## Lab Environment

- **Tool**: Cisco Packet Tracer
- **Network Type**: Simulated small-to-medium business network
- **Topology**: Pre-built network with multiple LANs and WAN connections

## Tools Used

- Cisco Packet Tracer

## Methodology

### Part 1: Identify common components of a network as represented in Packet Tracer.
I started by exploring the icon toolbar in Packet Tracer to understand the different device categories available. This helped me recognize which devices were intermediary (routers and switches) versus end devices (computers, servers).

**Question: List the intermediary device categories**

- Routers
- Switches
- Hubs
- Wireless Devices
- Security
- WAN Emulation

![Intermediary Devices](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Intermidiary%20devices.jpg)


**Question: Without entering into the internet cloud or intranet cloud, how many icons in the topology represent endpoint devices (only one connection leading to them)?**

**Answer:** There are **15** endpoint devices

![endpoint devices](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/end%20point%20devices.jpg)

**Question: Without counting the two clouds, how many icons in the topology represent intermediary devices (multiple connections leading to them)?**

**Answer:** There are **13** intermediary devices in the network

![Intermediary Device](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Multiple%20intermediary%20connection.jpg)

**Question: How many end devices are not desktop computers?**

**Answer:** There are **8** end devices that are not desktop computers.

![non desktop devices](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/non%20desktop%20end%20devices.jpg)


**Question: How many different types of media connections are used in this network topology?**

**Answer:** There are **2** which are:
- Cable
- Wireless

### Part 2: Explain the purpose of the devices

I examined the topology without entering the cloud icons to count actual physical devices in the network. This gave me a realistic view of what hardware would exist in a real business network.

**Question 2a: In Packet Tracer, only the Server-PT device can act as a server. Desktop or Laptop PCs cannot act as a server. Based on your studies so far, explain the client-server model.**

- The Client-Server model is a networking infrastructure that allows Server-PT to serve as a centralized system that stores, manages, and provides requested services and information to clients, while the Desktop and Laptop PCs are the clients that request and receive services and information from the server (Server-PT)

**Question 2b: List at least two functions of intermediary devices.**

- They connect end devices to networks and the internet.
- They also determine the route that the message should take through the network.

**Question 2c: List at least two criteria for choosing a network media type**

- The geographical area.
- The Speed required for data transmission.

### Part 3: Compare and contrast LANs and WANs

I identified which parts of the network were LANs (local area networks within buildings) versus WANs (wide area networks connecting different locations). This showed how businesses connect multiple sites.

**Question 3a: Explain the difference between a LAN and a WAN. Give examples of each**

- LAN (Local Area Network) is a network that connects users and networking devices to each other and the network in a small location, such as schools, laboratories, etc. Example: A computer laboratory that consists of multiple computers interconnected together.
- WAN (Wide Area Network) is a network that connects multiple LANs across cities, regions, and even continents. Example: A network connection of a financial bank with different branches situated at different locations across the country.

**Question 3b: In the Packet Tracer network, how many WANs do you see?**
- There are two WANs, which are: **Internet and Intranet (Cloud)**

![LAN & WAN](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/LAN%20%26%20WAN.jpg)

**Question 3c: How many LANs do you see?**

There are three (3), which are: 
- Central
- Branch
- Home Office

**Question 3d: The internet in this Packet Tracer network is overly simplified and does not represent the structure and form of the real internet. Briefly describe the Internet.**
- The internet is a collection of interconnected devices across the globe, allowing devices to share resources and information through standard protocols. In this Packet Tracer network, the internet, having its own clustered network is interconnecting both central, branch and home office connections.

**Question 3e: What are some of the common ways a home user connects to the internet?**

A home user may connect to the internet through:
- Cable
- Digital Subscriber
- Cellular Network

**Question 3f: What are some common methods that businesses use to connect to the internet in your area?**
- Satellite
- Metro Ethernet

## Challenge Question

**Question: Add an end device to the topology and connect it to one of the LANs with a media connection. What else does this device need to send data to other end users? Can you provide the information?**

- The device still needs to be configured with an IP address and subnetting address to receive and send data to other end users.

**Question: Is there a way to verify that you correctly connected the device?**

 - Yes, see the image below to see the New End Device connected to Switch (S2) through a copper straight-through cable

![Challenge 1](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Challenge%201.jpg)

**Question: Add a new intermediary device to one of the networks and connect it to one of the LANs or WANs with a media connection. What else does this device need to serve as an intermediary to other devices in the network?**

- The device still needs to be configured with an IP address and network interface before it can serve as an intermediary device.

![Challenge 2](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/Challenge%202.jpg)


**Question: Open a new instance of Packet Tracer. Create a new network with at least two LANs connected by a WAN. Connect all the devices.**

![new instances](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/new%20instance.jpg)

## Findings

**Device counts from topology analysis:**
- Intermediary device categories: [Routers, Switches, Hubs, Wireless Devices, Security, WAN Emulation]
- Endpoint devices: [15]
- Intermediary devices: [13]
- Non-desktop end devices: [8]
- Media connection types: [Cable and Wireless]
- WANs in topology: [Internet and Intranet (Cloud)]
- LANs in topology: [Central, Branch and Home Office]


## Key Takeaways

- Network topology visualization helps understand data flow before implementation
- Different device types serve specific purposes in network architecture
- LANs handle local traffic while WANs connect distant locations
- Packet Tracer provides a risk-free environment to explore network concepts
- Proper device selection depends on business requirements and scale

## Disclaimer

This lab was performed using Cisco Packet Tracer for educational purposes as part of CCNA coursework. All activities were conducted in a controlled simulation environment.
