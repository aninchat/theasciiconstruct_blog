---
title: "Cisco SDA and Security Part IV - micro-segmentation in SDA continued (some more)"
date: 2022-01-12T08:14:09+05:30
draft: false
tags: [sda, ise, sgt, micro-segmentation]
description: In this post, we will look at how to leverage SXP tunnels in ISE to achieve a specific use case.
---

## Introduction

Using SGTs, we were able to control communication between endpoints in the same VN. But what happens when traffic from these endpoints leaves the fabric? At the borders, the packet is decapsulated and since the outer header is stripped, along with the VXLAN header, we lose all SGT information.


So, to understand how we can control such kind of traffic, let’s introduce a corporate server in the shared segment with an IP address of 172.16.24.200 – our goal is to ensure that ODC users should not be able to talk to this corporate server.

![static1](/images/cisco/sda_security_3/security4_1.jpg)

There are several ways to do this, however, the most flexible way is via SXP tunnels. The idea is this – we need to allow the borders to do CTS role-based enforcement as well (which is not enabled by default in SDA) and pass IP to SGT mappings to the borders via an SXP tunnel between ISE and the borders.

## Setting up SXP tunnels

So, let’s take this step by step. The first goal is to bring up an SXP tunnel between Border1 and ISE.

To begin with, ensure SXP is enabled in ISE:

![static1](/images/cisco/sda_security_3/security4_2.jpg)

Make sure the appropriate interface is selected here – I’ve chosen gig1 since that is my fabric facing interface on ISE:

```
aninchat-ISE/admin# show interface
GigabitEthernet 0
        flags=4163  mtu 1500
        inet 10.104.152.28  netmask 255.255.255.0  broadcast 10.104.152.255
        inet6 fe80::20c:29ff:fe10:34b9  prefixlen 64  scopeid 0x20
        ether 00:0c:29:10:34:b9  txqueuelen 1000  (Ethernet)
        RX packets 1459508  bytes 251023644 (239.3 MiB)
        RX errors 0  dropped 977  overruns 0  frame 0
        TX packets 427767  bytes 145511393 (138.7 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 19  memory 0xfd3a0000-fd3c0000  

GigabitEthernet 1
        flags=4163  mtu 1500
        inet 172.16.24.11  netmask 255.255.0.0  broadcast 172.16.255.255
        inet6 fe80::20c:29ff:fe10:34c3  prefixlen 64  scopeid 0x20
        ether 00:0c:29:10:34:c3  txqueuelen 1000  (Ethernet)
        RX packets 51534  bytes 4799917 (4.5 MiB)
        RX errors 0  dropped 30  overruns 0  frame 0
        TX packets 51806  bytes 4141515 (3.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 16  memory 0xfd2a0000-fd2c0000
```

Once the SXP service is enabled, create a domain. Typically, this is not needed and you can use the default domain as well. But with a custom SXP domain, it is much easier to control which devices you are pushing IP to SGT mappings to.

![static1](/images/cisco/sda_security_3/security4_3.jpg)

Here, as you can see, I have created a domain called "sda borders”.

The next thing is to build the actual SXP tunnel itself. An important aspect of this to remember – for every VRF that you have, a separate SXP tunnel must be built since the IP to SGT mappings will be specific to VNs (VRFs). So, before we build the tunnel, let’s ensure that there are no problems in reachability. I want to build an SXP tunnel from VLAN 3014 on Border1 (which is in VRF Corp_VN) to the ISE IP address 172.16.24.11.

```
Border1#show run int vlan 3014
Building configuration...

Current configuration : 184 bytes
!
interface Vlan3014
 description vrf interface to External router
 vrf forwarding Corp_VN
 ip address 192.2.100.1 255.255.255.252
 no ip redirects
 ip route-cache same-interface
end

Border1#ping vrf Corp_VN 172.16.24.11 source 192.2.100.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.24.11, timeout is 2 seconds:
Packet sent with a source address of 192.2.100.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

Let’s first create the SXP tunnel on ISE, to Border1. This is where you input the SXP domain as well and using this domain, you can control which devices can get what update:

![static1](/images/cisco/sda_security_3/security4_4.jpg)

Once this is saved, configure matching parameters on Border1 as well:

```
Border1(config)#cts role-based enforcement
Border1(config)#cts role-based enforcement vlan-list 3014
Border1(config)#cts sxp enable
Border1(config)#cts sxp default password C1sc0123
Border1(config)#cts sxp connection peer 172.16.24.11 source 192.2.100.1 password default mode local listener vrf Corp_VN
```

Confirm that the SXP tunnel is up on both sides:

```
Border1#show cts sxp connections vrf Corp_VN
 SXP              : Enabled
 Highest Version Supported: 4
 Default Password : Set
 Default Source IP: Not Set
Connection retry open period: 120 secs
Reconcile period: 120 secs
Retry open timer is not running
Peer-Sequence traverse limit for export: Not Set
Peer-Sequence traverse limit for import: Not Set
----------------------------------------------
Peer IP          : 172.16.24.11
Source IP        : 192.2.100.1
Conn status      : On
Conn version     : 4
Conn capability  : IPv4-IPv6-Subnet
Conn hold time   : 120 seconds
Local mode       : SXP Listener
Connection inst# : 1
TCP conn fd      : 1
TCP conn password: default SXP password
Hold timer is running
Duration since last state change: 0:04:26:31 (dd:hr:mm:sec)


Total num of SXP Connections = 1

 0xFF913C6350 VRF:Corp_VN, fd: 1, peer ip: 172.16.24.11
cdbp:0xFF913C6350 Corp_VN <172.16.24.11, 192.2.100.1> tableid:0x5
```

Confirm the status on ISE as well:

![static1](/images/cisco/sda_security_3/security4_5.jpg)

The “Status” says ON, so we’re good on this front. Perfect, the tunnel is up on both sides!

## Creating the SGT and SGACL on DNAC

Remember, we still haven’t created any SGTs or SGACLs for this new corporate server. Let’s do that quickly from DNAC:

![static1](/images/cisco/sda_security_3/security4_6.jpg)

As you can see, we’ve created an SGT called “Corp_Servers” with a decimal value of 23 and assigned it to the VN “Corp_VN”.

Let’s create an SGACL now that denies communication between ODC_Users and Corp_Servers:

![static1](/images/cisco/sda_security_3/security4_7.jpg)

Once this is saved, make sure it syncs to ISE and you can see it in the ISE policy matrix as well:

![static1](/images/cisco/sda_security_3/security4_8.jpg)

## Creating a static IP to SGT mapping on ISE

Finally, it’s time to create a static IP to SGT mapping on ISE – this is what is referenced to push the SGACL to the Border. The IP address you put here is a /32 entry. Along with the IP address, select the SGT and the SXP domain as well. There is no need to select a device to deploy – the deployment happens on its own and we’ll understand how soon enough.

![static1](/images/cisco/sda_security_3/security4_9.jpg)

## Verification

Prior to saving this, quickly look at the SGTs and SGACLs present on Border1

```
Border1#show cts role-based permissions 
IPv4 Role-based permissions default:
        Permit IP-00
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE

Border1#show cts role-based sgt-map vrf Corp_VN all
%IPv6 protocol is not enabled in VRF Corp_VN
```

As you can see, there are no SGTs or SGACLs present currently. Let’s save the IP to SGT mapping now. After this, look at the same commands on Border1 again:

```
Border1#show cts role-based sgt-map vrf Corp_VN all
%IPv6 protocol is not enabled in VRF Corp_VN
Active IPv4-SGT Bindings Information

IP Address              SGT     Source
============================================
172.16.24.200           23      SXP

IP-SGT Active Bindings Summary
============================================
Total number of SXP      bindings = 1
Total number of active   bindings = 1


Border1#show cts role-based permissions 
IPv4 Role-based permissions default:
        Permit IP-00
IPv4 Role-based permissions from group 18:ODC_Users to group 23:Corp_Servers:
        Deny IP-00
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE
```

So, how did this really work behind the scenes? The SXP tunnel is used for the simple purpose of PUSHING an SGT to the device (Border1, in this case). When you save the IP to SGT mapping on ISE, the following debugs prove that the SXP tunnel is used to push this mapping to Border1:

```
*snip*

Apr 29 11:13:39.023: CTS-SXP-MSG:RCVD peer 172.16.24.11 readlen:28, datalen:0 remain:4096 bufp = 
FFA2C8DFF0: 0000001C 00000003 101004AC 10180B10  ...........,....
FFA2C8E000: 11020017 100B0520 AC1018C8 00000000  ....... ,..H....
FFA2C8E010: 00000000 00000000 00000000 00000000  ................
FFA2C8E020: 00000000 00000000 00000000 00000000  ................
FFA2C8E030: 00000000 00000000 00000000 00000000  ................
FFA2C8E040: 00000000 00000000 00000000 00000000  ................
FFA2C8E050: 00000000 00000000 00000000 00000000  ................
FFA2C8E060: 00000000 00000000 00000000 00000000  ................
FFA2C8E070: 00000000 00000000 00000000 00000000  ................
FFA2C8E080: 00000000 00000000 00000000 00000000  ................
FFA2C8E090: 00000000 00000000 00000000 00000000  ................
FFA2C8E0A0: 00000000 00000000 00000000 00000000  ................
FFA2C8E0B0: 00000000 00000000 00000000 00000000  ................
FFA2C8E0C0: 00000000 00000000 00000000 00000000  ................
FFA2C8E0D0: 00000000 00000000 00000000 00000000  ................
FFA2C8E0E0: 00000000 00000000 00000000 00000000  ................
FFA2C8E0F0:                                                      
Apr 29 11:13:39.050: CTS-SXP-MSG:sxp_handle_rx_msg_v2 <1>, Corp_VN<172.16.24.11, 192.2.100.1>
Apr 29 11:13:39.050: CTS-SXP-MSG:sxp_recv_update_v4 <1> peer ip: 172.16.24.11
Apr 29 11:13:39.052: CTS-SXP-INTNL:sxp_mdb_ip_sgt_entry_ip_dev_cmp one: 172.16.24.200, sgt 23:Corp_Servers, vrf 5, two: 172.16.24.200, sgt 50101, vrf 5
Apr 29 11:13:39.053: CTS-SXP-MDB:SXP MDB: Entry added ip 172.16.24.200 sgt 0x0017

*snip*
```

Once Border1 learns of a new SGT, it follows the same logic as our Edges did – it sends out an Access-Request to ISE, with an AV pair that lists this new SGT it learned. ISE looks at its SGACLs and if there is any SGACL with this SGT as the destination SGT, it responds back to Border1.

ISE Live Logs confirm this Access-Request with a username of CTSREQUEST. Notice that the NAD is Border1, as expcted:

![static1](/images/cisco/sda_security_3/security4_10.jpg)

A packet capture confirms the same process again:

![static1](/images/cisco/sda_security_3/security4_11.jpg)

The reply shows the SGACL that ISE returned:

![static1](/images/cisco/sda_security_3/security4_12.jpg)

## Summarizing the entire flow

This entire flow can be summarized as below:

![static1](/images/cisco/sda_security_3/security4_13.jpg)

As a final confirmation of what we wanted to achieve here, let’s ping from Host2 to the Corporate Server. The pings will and are accounted for as hardware denied packets:

```
// baseline before ping

Border1#show cts role-based counters  
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          29221      15351      0          0         
18      23      0          0          0          0          0          0         

// after ping 

Border1#show cts role-based counters 
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          30363      176300     0          0         
18      23      0          4          0          0          0          0   
```

This confirms the packets were dropped on the border. This also concludes our little micro-segmentation journey! I hope this was informative.
