---
title: Crazy debugging - the ARP bubble
date: 2020-06-10 23:14:18
thumbnail: "/images/arp_bubble.jpeg"
description: "How we screwed up the communication between VM and its Host big time.
ARP source learning caused a host scoped IP address to be used for outside communication."
tags:
 - debug
 - crazy debugging
 - networking
---

We had a requirement to collect a particular metric from each VMs (Virtual Machines) running in our cloud. The metric should be sent a central server periodically. Due to our VPC (Virtual Private Cloud) architecture, not all VMs can connect to the central server. So we decided to send the metric from each VMs to its own Host. And the Hosts will relay it to the central server. But we screwed it big time.

Before explaining the issue, let me brief about the topology and give you a simple introduction to ARP (Address Resolution Protocol).

## Architecture

Below diagram will help you understand the topology better.

![Architecture diagram](/images/arp_bubble_arch.png)

 * Each BM (Baremetal) or Host can able to host multiple VMs
 * VMs inside a Host are connected using a virtual interface (`veth*`) to Host's physical interface (`eth0`) via a virtual bridge (`br0`)
 * The virtual interface is implemented as TAP. A single virtual interface will look as `veth*` in the Host. The same interface will look as `eth0` inside VM. Lets not go deep into that now.
 * VMs can communicate with other VMs running on its own Host and to its Host as well via `br0`
 * The physical interface of each Host is connected to a TOR (Top Of the Rack) switch
 * So all VMs can connect to all other VMs and BMs in the rack (Ignore VPC rules now)
 * VM will think itself as standalone machine and consider rest all machines (including its own Host) as outside machines connected via network

From now onwards I'll keep the image as a reference. *Refer it for name of each BMs, VMs, network interfaces and their IP, MAC addresses.*

## ARP - Address Resolution Protocol

*If you know about ARP, skip this section.*

ARP is a layer-2 protocol. Its used for next hop communication.

Every network device (node) has a globally unique 64-bit address called MAC address (Media Access Control address). For any network device to send a packet to another network device, it should know the MAC address of that device. Because the ethernet header of the packet should contain the MAC address of the next-hop device (immediate next device in the path - not including transparent devices like `br0` or TOR switch).

For example, when VM0 wants to send a packet to VM1 it should know the MAC address of VM1's network interface, in our case `veth1`.

As admin can't program the MAC address of all connected interfaces manually, we need some better mechanism. ARP is that better mechanism. Its very simple. Whenever a  node wants to send a packet to an IP address but it doesn't know corresponding MAC address, it will send a ARP Request. Upon reception, the destination node will respond with and ARP Reply telling its MAC address to the first node.

Let me explain the details now.

Each ARP packet will have four fields
 1. Source IP
 2. Source MAC
 3. Destination IP
 4. Destination MAC

Wild-card MAC: 00:00:00:00:00:00

ARP Request will have Source IP and Source MAC as sender's IP and MAC. Destination IP as destination node's IP and the Destination MAC will be wild-card MAC. ARP Reply will have Source IP and Source MAC as Sender's IP and MAC and Destination IP and Destination MAC as ARP Requester's IP and MAC.

The ethernet frame around the ARP Request should have a MAC address. As the destination MAC is unknown these packets will be sent to the wild-card MAC. Packets sent to wild-card MAC will be received by all connected machines. This called broadcast. When the destination is known, these packets will be sent to that particular MAC and they will reach that particular node only. These are called unicast packets.

<b>example - 1:</b>
 Lets say VM-0 wants to send a packet to VM-3. VM-0 knows the IP address of VM-3. Otherwise it can't construct the packet itself. And VM-0 doesn't know the MAC address of VM-3. So the following sequence of action will happen.

*NOTE: If you wonder how VM-1 knows the IP of VM-3, read about DNS. Sometimes it will be direct hard-coded IP or user manually enters the IP like an argument to ping request.*

VM-0 will broadcast an ARP Request with following values.

| Field           | Value             |
| ---             | ---               |
| Source IP       | 10.0.0.10         |
| Source MAC      | 00:00:00:00:10:00 |
| Destination IP  | 10.0.1.10         |
| Destination MAC | 00:00:00:00:00:00 |

This request will be received and ignored by VM-1, VM-4, BM-1 and BM-2 as the IP doesn't belong to them. But VM-3 will respond a ARP Reply with following values. The response will be unicast to VM-0's MAC.

| Field           | Value             |
| ---             | ---               |
| Source IP       | 10.0.1.10         |
| Source MAC      | 10:00:00:00:10:00 |
| Destination IP  | 10.0.0.10         |
| Destination MAC | 00:00:00:00:10:00 |

After receiving the ARP Reply VM-0 sends the actual packet to VM-3's MAC address. Also VM-0 will store this information for a period of time. This IP-MAC store is called ARP cache. This cache will be referred for any future communications. It saves a lot of ARP packet flow.

Read more about ARP at [Wikipedia](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)

## Problem statement

Our requirement is collecting a particular metric from each VM and store it in a central database. What metric we are going to collect doesn't matter here. We had below constraints.

 * VMs can't directly send the data to the central server
 * VMs will not know the IP address of their Host
 * Host doesn't know the IP address of VMs it hosts.
 * Host can't pull data from VMs. Because VMs are customer facing. Keeping a port open in VMs is serious security violation.

## Design
So we designed our metric collection model as below
 * We'll add one more IP to each Host's physical interface - 169.254.1.1. Let's call it as LOCAL_IP
 * LOCAL_IP is link scoped IP. It will not be used for outside communication.
 * Add a `ebtables` entry to all BMs - To respond to ARP Requests for 169.254.1.1 with Host's MAC address
 * A simple script will run inside the VM which collects that metric. And it will send it to LOCAL_IP periodically. Name it as VM_SCRIPT.
 * A simple web-server is running in each Host to collect the data sent by VM_SCRIPT and relay it to central server. Name it as METRIC_COLLECTOR

All look good rite? Lets examine a scenario. How the metric is collected from VM-1?

<b>example-2</b>
 1. VM_SCRIPT will collect the metric. And it will send the data to LOCAL_IP
 2. As MAC address of LOCAL_IP is not known, VM-1 will broadcast an ARP Request for LOCAL_IP
 3. BM1 will respond the ARP Request with eth0's MAC address. Thanks to the `ebtables` rules added.
 4. VM-1 will send the metric data packet to BM1's `eth0`'s MAC address
 5. METRIC_COLLECTOR running at BM1 will receive the data and relay it central server

But things didn't turn as expected. Actually it was a complete mess.

## The catch - ARP source learning

*If you know about ARP source learning skip this section.*

Let me explain you about a simple feature of ARP. Being smart, the ARP designers introduced something called source learning. Its nothing but learn the MAC address from an ARP Request itself.

When a node receives an ARP Request it means that sender wants to communicate with this node. So most probably this node need to send some data back to the sender. In that case, the first node needs to know the MAC of second node. To save an ARP Request-Reply latency in future, the first node will store the MAC address of second node from the ARP Request itself.

In our example-1, VM-3 will store the MAC of VM-0 when it receives ARP Request from VM-0.

Source learning will happen only in two scenarios.
 a) If the Destination IP is my IP
 b) If Source IP is already in my ARP cache. I'll update the MAC address and reset cache expiry timer.

## The climax

All the details above are give only explain this problem. What actually happened was Hosts started receiving metric data from random VMs. I'll continue example-2 to explain the problem.

 4. VM_SCRIPT in VM-1 will send a TCP_SYN (read about TCP handshake) to LOCAL_IP. The MAC address in the ethernet header will be the MAC address received at step-3
 5. Upon receiving TCP_SYN from VM-1, BM-1 will initiate a TCP_ACK. But for that it needs to know the MAC address of VM-1.
 6. <b>BM-1 didn't do source learning. Because `ebtables` rules responded the ARP request. The packet didn't reach actual ARP stack.</b> - may be a bug in `ebtables` code.
 7. So now BM-1 will broadcast a ARP Request with following fields.

| Field           | Value             |
| ---             | ---               |
| Source IP       | 169.254.1.1       |
| Source MAC      | 00:00:00:00:00:01 |
| Destination IP  | 10.0.0.11         |
| Destination MAC | 00:00:00:00:00:00 |

 8. This ARP Request will reach VM-0, VM-1, VM-3 and VM-4 as they are in same network. The LOCAL_IP is host-scoped IP won't be used for outside communication. But this is ARP Request, no IP layer involves here (end of the day everything is code rite)
 9. And all VMs store MAC - 00:00:00:00:00:01 for LOCAL_IP. And this MAC will be used for next communication to LOCAL_IP. (This will happen only if LOCAL_IP is already in ARP cache.)
 10. From now onwards, all VMs including VM-3 and VM-4 will start sending data to BM-1

## Demo
I'll explain it with a ping request. The sequence of action will be,
 1. VM-1 pings BM1 using its LOCAL_IP
 2. BM1 will send ARP Request for VM-1
 3. VM-3 will learn the MAC address of BM1
 4. Further communication from VM-3 to LOCAL_IP will reach BM1 instead of BM2

Lets see IP, MAC and `ebtables` rules in each machine.

**VM-1**
IP: <b>10.51.162.74</b>
MAC: <b>02:01:0a:33:a2:4a</b>
```sh
VM-1 $ ip address show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 02:01:0a:33:a2:4a brd ff:ff:ff:ff:ff:ff
    inet 10.51.162.74/20 brd 10.51.175.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1:aff:fe33:a24a/64 scope link
       valid_lft forever preferred_lft forever
VM-1 $
```

**BM1**
IP: <b>169.254.1.1, 10.51.162.3</b>
MAC: <b>98:03:9b:a0:98:2a</b>
```sh
BM1 $ ip address show dev enp94s0f0
4: enp94s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc htb state UP group default qlen 1000
    link/ether 98:03:9b:a0:98:2a brd ff:ff:ff:ff:ff:ff
    inet 169.254.1.1/32 scope host enp94s0f0
       valid_lft forever preferred_lft forever
    inet 10.51.162.3/20 brd 10.51.175.255 scope global enp94s0f0
       valid_lft forever preferred_lft forever
BM1 $ sudo ebtables -t nat -L
Bridge table: nat

Bridge chain: PREROUTING, entries: 2, policy: ACCEPT
-p ARP -i vnet+ --arp-op Request --arp-ip-dst 169.254.1.1 -j arpreply --arpreply-mac 98:3:9b:a0:98:2a
-p ARP -i veth+ --arp-op Request --arp-ip-dst 169.254.1.1 -j arpreply --arpreply-mac 98:3:9b:a0:98:2a

Bridge chain: OUTPUT, entries: 0, policy: ACCEPT

Bridge chain: POSTROUTING, entries: 0, policy: ACCEPT
BM1 $
```

**VM-3**
IP: <b>10.51.163.7</b>
MAC: <b>02:01:0a:33:a3:07</b>
```sh
VM-3 $ ip a s dev `eth0`
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:01:0a:33:a3:07 brd ff:ff:ff:ff:ff:ff
    inet 10.51.163.7/20 brd 10.51.175.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1:aff:fe33:a307/64 scope link
       valid_lft forever preferred_lft forever
VM-3 $
```

**BM2**
IP: <b>169.254.1.1, 10.51.162.28</b>
MAC: <b>98:03:9b:a9:78:ca</b>
```sh
BM2 $ ip a s dev enp94s0f0
48: enp94s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc htb state UP group default qlen 1000
    link/ether 98:03:9b:a9:78:ca brd ff:ff:ff:ff:ff:ff
    inet 169.254.1.1/32 scope host enp94s0f0
       valid_lft forever preferred_lft forever
    inet 10.51.162.28/20 brd 10.51.175.255 scope global enp94s0f0
       valid_lft forever preferred_lft forever
BM2 $ sudo ebtables -t nat -L
Bridge table: nat

Bridge chain: PREROUTING, entries: 2, policy: ACCEPT
-p ARP -i vnet+ --arp-op Request --arp-ip-dst 169.254.1.1 -j arpreply --arpreply-mac 98:3:9b:a9:78:ca
-p ARP -i veth+ --arp-op Request --arp-ip-dst 169.254.1.1 -j arpreply --arpreply-mac 98:3:9b:a9:78:ca

Bridge chain: OUTPUT, entries: 0, policy: ACCEPT

Bridge chain: POSTROUTING, entries: 0, policy: ACCEPT
BM2 $
```

Let's ping LOCAL_IP from VM-1.
To trigger a ARP Request from BM1, first I should delete the ARP cache entry in BM1 for VM-1's IP if present.
```sh
BM1 $ sudo arp | grep 10.51.162.74
10.51.162.74             ether   02:01:0a:33:a2:4a   C                     enp94s0f0
BM1 $ sudo arp -d 10.51.162.74
BM1 $ sudo arp | grep 10.51.162.74
10.51.162.74                     (incomplete)                              enp94s0f0
BM1 $
```

And make sure in VM-3 there is an ARP cache entry for LOCAL_IP. Then only the source learning will happen.
*Please note that the MAC address against LOCAL_IP is MAC address of BM2.*
```sh
VM-3 $ sudo arp | grep 169.254.1.1
169.254.1.1              ether   98:3:9b:a9:78:ca   C                     eth0
VM-3 $
```

Now ping LOCAL_IP from VM-1. In parallel I start packet capturing in VM-3.
```sh
VM-1 $ ping -c1 169.254.1.1
PING 169.254.1.1 (169.254.1.1) 56(84) bytes of data.
64 bytes from 169.254.1.1: icmp_seq=1 ttl=64 time=0.253 ms

--- 169.254.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.253/0.253/0.253/0.000 ms
VM-1 $
```

VM-3 packet capturing status. And ARP cache after the ping request.
*Please note now the MAC against LOCAL_IP is MAC of BM1*
```sh
VM-3 $ sudo tshark -f "arp src 169.254.1.1" -i eth0
Running as user "root" and group "root". This could be dangerous.
Capturing on 'eth0'
    1 0.000000000 Mellanox_a0:98:2a ? Broadcast    ARP 60 Who has 10.51.162.74? Tell 169.254.1.1
^C1 packet captured
VM-3 $ sudo arp | grep 169
169.254.1.1              ether   98:03:9b:a0:98:2a   C                     eth0
VM-3 $
```

From now onwards all communication from VM-3 to LOCAL_IP will reach BM-1 not BM-2.
A ping request for LOCAL_IP from VM-3,
```sh
VM-3 $ ping -c1 169.254.1.1
PING 169.254.1.1 (169.254.1.1) 56(84) bytes of data.
64 bytes from 169.254.1.1: icmp_seq=1 ttl=64 time=0.205 ms

--- 169.254.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.205/0.205/0.205/0.000 ms
VM-3 $
```

Reaches BM-1 instead of BM-2
```sh
BM1 $ sudo tshark -f "ip src 10.51.163.7" -i enp94s0f0
Running as user "root" and group "root". This could be dangerous.
Capturing on 'enp94s0f0'
    1 0.000000000  10.51.163.7 ? 169.254.1.1  ICMP 98 Echo (ping) request  id=0x7fb1, seq=1/256, ttl=64
^C1 packet captured
BM1 $
```

## Fix
We fixed it with `arptables`. We added a rule to all bare-metals that no external ARP will be forwarded by any of its interfaces (both physical and virtual). Any ARP packets with source IP as LOCAL_IP and MAC not as current bare-metal's MAC will be dropped.
```sh
BM1 $ sudo arptables -L
Chain INPUT (policy ACCEPT)

Chain OUTPUT (policy ACCEPT)

Chain FORWARD (policy ACCEPT)
-j DROP -s 169.254.1.1 ! --src-mac 98:03:9b:a0:98:2a
BM1 $
```
This rule prevents the VMs from receiving ARP Requests from neighbour bare-metals.
