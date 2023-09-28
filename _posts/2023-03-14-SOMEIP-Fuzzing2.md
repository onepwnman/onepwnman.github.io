---
layout: post
title: "How to fuzz SOME/IP protocol - How did I get my name on the BMW Hall of Fame (Part 2)"
tags: [BMW, SOME/IP, Fuzzing]
reference: 
  - "vsomeip-in-10-minutes" : https://github.com/COVESA/vsomeip/wiki/vsomeip-in-10-minutes#prep
---

In the previous [post](https://onepwnman.github.io/SOMEIP-Fuzzing), we learned what is SOME/IP protocol and how the SOME/IP header is structured and how payload serialization works. We also talk about that the SOME/IP-SD protocol is used to obtain the information that goes into the SOME/IP header. From now on, let's take a look at what the SOME/IP-SD protocol is and how to make use of SOME/IP-SD protocol to fuzz SOME/IP protocol.

<br>
## What is SOME/IP-SD Protocol?
SOME/IP-SD is a protocol implemented on top of the SOME/IP layer. It is used to find service instances in a SOME/IP network, detect whether a service instance is running, and implement publish/subscribe processing.

This is done through so-called offer messages. In other words, each device broadcasts(multicasts) a message containing all the services that it provides on this device. Client applications can also send "find messages" if they need a service that is not currently provided. Other SOME/IP-SD messages can be used to publish/subscribe to event groups.

(SOME/IP-SD messages are transmitted over UDP port 30490 in default settings.)

<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/someip-sd-protocol-layers.png" style="padding: 0;margin:0;">
</p>

<br>
###  SOME/IP-SD packet structure
The SOME/IP header area of SOME/IP-SD has fixed values as figure below. Since the message type is 0x02, which is Notification, SOME/IP-SD messages do not expect a response.
<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/someip-sd-header-structure.png" style="padding: 0;margin:0;">
</p>

  - **Message ID (Service ID/Method ID)** [32 bit]: 0xFFFF 8100
  - **Protocol Version** [8 bit]: 0x01
  - **Interface Version** [8 bit]: 0x01
  - **Message Type** [8 bit]: 0x02 (Notification)
  - **Return Code** [8 bit]: 0x00


The first flag of the SOME/IP-SD Flags field (highest order bit) is called the Reboot Flag.
The Reboot Flag of the SOME/IP-SD Header is set to 1 for all messages after reboot until the Session ID in the SOME/IP header wraps around and starts with 1 again. After this wrap-around, the Reboot Flag is set to 0.
The second flag of the SOME/IP-SD Flags (second highest order bit) is called the Unicast Flag.
The Unicast Flag is a remnant of historical SOME/IP versions and is only kept for compatibility. Its use is very limited outside of this context.


A single SD message supports multiple entries, which are used to synchronize the state of a service instance or for publish/subscribe handling.

### Service Entry / Eventgroup Entry
<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/entries.png" style="padding: 0;margin:0;">
</p>

<br>

**Offer service** messages are used by service providers to advertise their services to the network and **Find service** messages are used by service consumers to search for services in the network.
Following image is a description of each element of a service entry.
<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/service-entry.png" style="padding: 0;margin:0;">
</p>

- **Type Field** [uint8]: FindService (0x00), OfferService (0x01), StopOfferService (0x01, TTL=0x00)
- **Index 1st Option** [uint8]: Index of 1st option in the option array.
- **Index 2nd Option** [uint8]: Index of 2nd option in the option array.
- **Number of Options 1** [uint4]: The number of options the 1st option run uses. 0 means no option.
- **Number of Options 2** [uint4]: The number of options the 2nd option run uses. 0 means no option.
- **Service-ID** [uint16]: The Service ID of the Service or Service-Instance.
- **Instance ID** [uint16]: The Service Instance ID of the Service Instance. 0xFFFF means all service instances.
- **Major Version** [uint8]: The major version of the service (instance).
- **TTL** [uint24]: The lifetime of the entry in seconds.
- **Minor Version** [uint32]: The minor version of the service.

<br>
An Eventgroup Entry is 16 bytes long and include the following fields, in this order
<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/eventgroup-entry.png" style="padding: 0;margin:0;">
</p>

- **Type Field** [uint8]: Subscribe (0x06), StopSubscribeEventgroup (0x06, TTL=0),  SubscribeAck (0x07), SubscribeEventgroupNack (0x07, TTL=0).
- <span style="color:gray"> Index of first option run [uint8], Index of second option run [uint8]: same with service entry </span>
- <span style="color:gray"> Number of Options 1 [uint4], Number of Options 2 [uint4]: same with service entry </span>
- <span style="color:gray"> Service-ID [uint16]: same with service entry
- **Instance ID** [uint16]: The Service Instance ID shall not be set to 0xFFFF for any Instance. </span>
- <span style="color:gray"> Major Version [uint8], TTL [uint24]: same with service entry </span>
- **Reserved** [uint12]: Shall be set to 0x000.
- **Counter** [uint4]: Is used to differentiate identical Subscribe Eventgroups of the same subscriber.  Set to 0x0 if not used.
- **Eventgroup ID** [uint16]: Transports the ID of an Eventgroup.

### Options formet (IPv4 Endpoint)
<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/options.png" style="padding: 0;margin:0;">
</p>
This includes for instance the information how a service instance is reachable (IP-Address, Transport Protocol, Port Number).
Options are used to transport additional information to the entries.
<br>


## SOME/IP-SD message analysis and fuzzing
I was provided with a simulation environment of MGU22 by the BMW private bug bounty program. The simulation environment is configured for Raspberry Pi, and the QEMU on the Raspberry Pi simulates the MGU22 environment. A dummy binary running on the Raspberry Pi simulates other controllers connected to MGU22, such as LIDAR, bodyECU, and chassisECU. The environment simulates internal SOME/IP communication and the entire in-vehicle system through communication between the dummy binary and QEMU (MGU22).

For SOME/IP, a network in the 160.48.199.xx range was used. In the case of the MGU22 controller, the IP address 160.48.199.99 was assigned, and body.caraccess was assigned IP address 160.48.199.16, body was assigned 160.48.199.64, and chassis was assigned 160.48.199.93. Other IP addresses, except for 160.48.199.99, are bound to dummy binaries outside of QEMU machine


##### Binding Table

| Address | ECU |
| --- | --- |
| 160.48.199.6 | dass |
| 160.48.199.16 | caraccess |
| 160.48.199.20 | UPCP_Safety |
| 160.48.199.34 | dassproposal |
| 160.48.199.64 | body |
| 160.48.199.72 | LIDAR_D_H |
| 160.48.199.73 | LIDAR_D_V |
| 160.48.199.74 | LIDAR_D_L |
| 160.48.199.75 | LIDAR_D_R |
| 160.48.199.76 | FRR_D_VL |
| 160.48.199.77 | FRR_D_VR |
| 160.48.199.78 | FRR_D_HL |
| 160.48.199.79 | FRR_D_HR |
| 160.48.199.93 | chassis |
| 160.48.199.96 | interiorillumination |
| 160.48.199.97 | infotainment |
| 160.48.199.98 | sapoverip |
| 160.48.199.121 | infotainment |
| 160.48.199.125 | testability |

In the Test Bench environment, it was confirmed that the network of the same range as the virtual environment and the same IP address are used. However, in the case of a real vehicle, each IP address of the dummy binary is bound to a different ECU. In order to connect to the vehicle network in a real vehicle environment and analyze it, the Ethernet interface address of the PC must be changed to an unallocated 160.48.199.xx range and the Ethernet base speed must be changed using a 100BASE-T1 switch. This process is required because the base speed of the vehicle network is different from the Ethernet network of the PC that we commonly use.

Since our fuzzing target IP is 160.48.199.99, which is the MGU22, we can consider two cases.
**The first is fuzzing the service offered by MGU22. (request/response)**
**The second is to tamper with the event messages sent to MGU22 by the services that MGU22 subscribes to.**

Also, message hooking can be used if a connection has already been established between MGU22 and the controller.
In the case connection between the dummy binary (ECUs) and QEMU (MGU22) is disconnected, the simulation will end, so we can only modify the messages sent from IP 160.48.199.16 (body.access) to 160.48.199.99 by hooking the message for fuzzing.

Using a tool like Frida, you can fuzz the SOME/IP communication between other controllers and the MGU22 by hooking the messages sent from the dummy binary to QEMU.

In a real test bench setup environment, you can remove one controller from the network, then send the SOME/IP messages from the PC to the MGU22 to simulate the normal system state. 

For example, if you remove the left lidar with the address 160.48.199.74 from the SOME/IP network, then connect the PC to the SOME/IP network and send the packets sent by the left lidar to MGU22, you can simulate a virtual lidar with the PC. You can fuzz by modifying the packets, and this is a common way to fuzz CAN or Ethernet-based vehicle networks. 

Let's take a look at the actual SOME/IP-SD message pcap capture now. Wireshark supports the SOMEIP schema from version 3.2.0.

<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/offer-service.png" style="padding: 0;margin:0;">
</p>
The details of the SOME/IP offer service message are as follows. An offer service message is multicasted to the addresses in the network from the address 160.48.199.99. The packet content uses the SOME/IP header as described above, uses the Message ID (Service ID/Method ID) of 0xFFFF 8100, and uses the fixed values of protocol version, interface version 0x1, message type 0x02 notification, and return code 0x00.

The following is a detailed description of the SOME/IP-SD payload.

<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/offer-service-detail.png" style="padding: 0;margin:0;">
</p>

The message type is Offer Service. Since there is one options array, the number of opts is 1. The index 1st option and index 2dn option both point to the 0th option array. The service id is 0xb063 and the major version matches the interface version.

The index of the options array in the offer service entry of the corresponding SOME/IP-SD message is 0, so it corresponds to the 0th option array.
The options array contains IP addresses, protocol types, and port numbers.

Therefore, the summary of the offer service message above is that the IP address 160.48.199.99 is providing a service corresponding to service ID 0xb063 on TCP port 32501.
In the case of MGU22, offer service was continuously sent when the power was turned on, but if there is no offer service, you can also know the service ID of the service provided by sending a find service message.

In the case of service ID 0xb063, as you do not know the method ID and payload structure, you need to fuzz the method ID and payload randomly. In this case, the payload must be serialized to random data types

<br>
The second method is to find the service ID that MGU22 subscribes to and send a modified event message.

<p align="center">
  <img alt="Fuzzing SOME/IP" src="/assets/images/someip-fuzzing2/subscribe-event.png" style="padding: 0;margin:0;">
</p>
If you can capture the event packet sent to the address 160.48.199.99 as shown in the figure, you can know the structure of the method ID and payload, making fuzzing easier.

<br>
<br>
### DEMO
The following is an example video of a vulnerability found through fuzzing.
If a process is killed, the recovery manager will restart the process or the MGU22 may reboot.
In the case of a head unit, the screen often goes black when a crash occurs, but in the case of a vehicle control unit without a display, it is difficult to visually identify a crash that occurs through fuzzing, so it is also very important to decide how to detect a crash.

Most control units use the CAN protocol today, so there are several detection methods through CAN protocol, such as checking DTCs when a crash occurs or checking the response of CAN request messages.

<iframe width="720" height="480" src="/assets/images/someip-fuzzing2/telematics_getValue_crash_demo.mp4" title="MGU22 Crash DEMO" frameborder="0" allow="clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>