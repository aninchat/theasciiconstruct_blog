---
title: "SDA and Wireless Part II - AP onboarding"
date: 2022-01-13T13:52:50+05:30
draft: false
tags: [sda, wireless]
description: In this post, we look at how an Access Point is onboarded in a SD-Access fabric.
---

## Introduction and topology

This post will continue to build on the topology that was used for Part I of this series.

![static1](/images/cisco/sda_wireless_2/wireless2_1.jpg)

## IP pools for access points

The IP addressing scope for APs falls within the pre-created INFRA_VN in fabric. This VN is not an overlay – it runs within the global routing table itself. Once the IP addressing is designed, define this within the INFRA_VN in your fabric:

![static1](/images/cisco/sda_wireless_2/wireless2_2.jpg)

This does several things (which is commonly done whenever you add an IP pool for any of your VNs):

It creates an instance ID in LISP for this (for both IPv4 and Ethernet) and creates a dynamic EID database-mapping for the IP range within the IPv4 service and the L2 VLAN (along with a database-mapping for any mac address) within the Ethernet service:

```
Edge2#show run | sec router lisp
router lisp

 *snip*

 !
 instance-id 4097
  remote-rloc-probe on-route-change
  dynamic-eid 192_2_12_0-INFRA_VN
   database-mapping 192.2.12.0/24 locator-set rloc_80a79d36-5b68-462f-b2d5-4ad61f5ab694
   exit-dynamic-eid
  !
  service ipv4
   eid-table default
   exit-service-ipv4
  !
  exit-instance-id
 !
instance-id 8191
  remote-rloc-probe on-route-change
  service ethernet
   eid-table vlan 1024
   database-mapping mac locator-set rloc_80a79d36-5b68-462f-b2d5-4ad61f5ab694
   exit-service-ethernet
  !
  exit-instance-id
 !

*snip*
```

It also creates a L2 VLAN and a corresponding SVI, assigning it an IP address that is specified as the default gateway for this IP pool in your DNAC design settings. LISP mobility is configured under this to allow for dynamic EIDs to be learnt.

```
Edge2#show run int vlan1024
Building configuration...

Current configuration : 285 bytes
!
interface Vlan1024
 description Configured from Cisco DNA-Center
 mac-address 0000.0c9f.f45f
 ip address 192.2.12.1 255.255.255.0
 ip helper-address 192.2.201.224
 no ip redirects
 ip route-cache same-interface
 no lisp mobility liveness test
 lisp mobility 192_2_12_0-INFRA_VN
end
```

Additionally, on the control-plane nodes, it creates corresponding instance IDs in LISP as well adding this IP range to be accepted as an EID within the site cache:

```
Border2#show run | sec router lisp
router lisp
 
*snip*

!
 instance-id 4097
  remote-rloc-probe on-route-change
  service ipv4
   eid-table default
   map-cache 192.2.12.0/24 map-request
   route-export site-registrations
   distance site-registrations 250
   map-cache site-registration
   exit-service-ipv4
  !
  exit-instance-id
!
site site_uci
  description map-server configured from apic-em
  authentication-key 7 08344F47
  eid-record instance-id 4097 192.2.12.0/24 accept-more-specifics
  eid-record instance-id 4099 192.2.21.0/24 accept-more-specifics
  eid-record instance-id 8190 any-mac
  eid-record instance-id 8191 any-mac
  exit-site
!

*snip*
```

It also creates a /32 loopback on the borders with the same anycast IP address as the edges for this SVI.

Lastly, this is advertised and aggregated in BGP to propagate this prefix to upstream neighbors:

```
Border2#show run | sec bgp
router bgp 65003
 bgp router-id interface Loopback0
 bgp log-neighbor-changes
 bgp graceful-restart
 neighbor 192.2.100.18 remote-as 65002
 neighbor 192.2.100.18 update-source Vlan3018
 !
 address-family ipv4
  bgp aggregate-timer 0
  network 192.2.12.1 mask 255.255.255.255
  network 192.2.99.69 mask 255.255.255.255
  aggregate-address 192.2.12.0 255.255.255.0 summary-only

*snip* 
```

If you look at the ‘show wireless fabric summary’ again now, you will see a new mapping in there for this IP pool that was just added to the INFRA_VN. This will also tell you what the corresponding L3 and L2 VNIDs are for this pool.

```
9800-WLC2#show wireless fabric summary 

Fabric Status      : Enabled


Control-plane: 
Name                             IP-address        Key                              Status
--------------------------------------------------------------------------------------------
default-control-plane            192.2.99.65       uci                              Up   
default-control-plane            192.2.99.69       uci                              Up   


Fabric VNID Mapping:
  Name               L2-VNID        L3-VNID        IP Address             Subnet        Control plane name
----------------------------------------------------------------------------------------------------------------------
  192_2_12_0-INFRA_VN     8191           4097           192.2.12.0          255.255.255.0 
```

## AP onboarding

### Acquiring an IP address

Everything is now in place for APs to be onboarded. The AP boots up as any other client and acquires an IP address via DHCP. The DHCP scope has option 43 configured, which defines the WLC IP address the AP should reach out to. 

```
Fusion#show run | sec ip dhcp
ip dhcp excluded-address 192.2.12.1

*snip*

ip dhcp pool IPP_OV_Infra
 network 192.2.12.0 255.255.255.0
 default-router 192.2.12.1 
 domain-name cisco.com
 dns-server 72.163.128.140 
 option 43 hex f104.ac15.0103
```

A DHCP offer (in the packet capture below) from the DHCP server shows the IP address that the AP is being allocated and option43 within the offer: 

![static1](/images/cisco/sda_wireless_2/wireless2_3.jpg)

## AP registration by fabric edge

Once the AP is up and starts sending some traffic, the edge will register the APs hardware mac address and IP address as EIDs to the control-plane(s): 

![static1](/images/cisco/sda_wireless_2/wireless2_4.jpg)

This can also be seen from the below packet captures. The first image shows the hardware mac registration while the second image shows the IP address registration:

![static1](/images/cisco/sda_wireless_2/wireless2_5.jpg)
![static1](/images/cisco/sda_wireless_2/wireless2_6.jpg)

## AP registration with WLC

At the same time, the AP (now with an IP address of 192.2.12.18) sends a discover to the WLC and gets a discover response. Post this, it builds a DTLS session with the controller and sends a join once the session is up. It expects to receive a join response back and once the process completes the AP is considered registered to the WLC. 

This AP join process can be seen in the below capture (joins will not be seen since that is post DTLS bring-up and is encrypted):

![static1](/images/cisco/sda_wireless_2/wireless2_7.jpg)

So, originally, when the AP is coming up, your CPs will see the edge as the RLOC for both the L2 and L3 instances, since the edge will register this with the CPs (as seen earlier via the packet captures). On the CPs, you can confirm this with the below commands:

```
// for service IPv4 

Border1#show lisp instance-id 4097 ipv4 server
LISP Site Registration Information
* = Some locators are down or unreachable
# = Some registrations are sourced by reliable transport

Site Name      Last      Up     Who Last             Inst     EID Prefix
               Register         Registered           ID       
site_uci       never     no     --                   4097     192.2.12.0/24
               00:14:55  yes#   192.2.99.68:12318    4097     192.2.12.18/32

// for service ethernet

Border1#show lisp instance-id 8191 ethernet server
LISP Site Registration Information
* = Some locators are down or unreachable
# = Some registrations are sourced by reliable transport

Site Name      Last      Up     Who Last             Inst     EID Prefix
               Register         Registered           ID       
site_uci       never     no     --                   8191     any-mac
               00:15:35  yes#   192.2.99.68:12318    8191     0000.0c9f.f45f/48
               00:15:05  yes#   192.2.99.68:12318    8191     380e.4d5b.0d40/48
               1w1d      yes#   192.2.99.68:12318    8191     7486.0b05.7e5c/48
```

However, there needs to be some way to inform the fabric infrastructure that what is connected to the edge is not just another client but an AP. LISP is leveraged for this. 

### Building a VXLAN tunnel to the AP

Once the AP registers with the WLC, the WLC sends a LISP map request to the control-plane nodes for this APs IP address. This is sent because at this point in time, the WLC has no idea what edge the AP is connected to (specifically, in LISP terms, what RLOC the AP is connected to). It can glean this information by sending a map request for the AP. 

![static1](/images/cisco/sda_wireless_2/wireless2_8.jpg)

This can also be visualized like below:

![static1](/images/cisco/sda_wireless_2/wireless2_9.jpg)
![static1](/images/cisco/sda_wireless_2/wireless2_10.jpg)

The following packet capture shows the map reply packet:

![static1](/images/cisco/sda_wireless_2/wireless2_11.jpg)

When the WLC receives this LISP map reply, it gathers the RLOC information from it and immediately sends another LISP message to the control-plane(s) with a LISP packet type of 31. 

This includes the radio mac of the AP and necessary encoding that allows this to be further propagated to the edge where the AP is connected. Wireshark cannot decode these packets since there are no dissectors available.

A packet capture for this:

![static1](/images/cisco/sda_wireless_2/wireless2_12.jpg)

This can also be visualized as below:

![static1](/images/cisco/sda_wireless_2/wireless2_13.jpg)

The CPs then send a LISP message (again, a LISP packet type of 31) to the RLOC (Edge1, in this case), along with the required meta-data to build a VXLAN tunnel to the AP. The following debug from the edge confirms this: 

```
*snip*

005333: Dec  1 06:59:04.869: [MS]  LISP: Session VRF default, Local 192.2.99.68, Peer 192.2.99.65:4342, Role: Active, State: Up, Received reliable registration message wlc mapping-notification for IID 8191  EID 6cb2.ae5a.eac0/48  (RX 0, TX 0).
005334: Dec  1 06:59:04.869: [XTR] LISP-0: MAC Map Server IID 8191 192.2.99.65, WLC notification for EID 6cb2.ae5a.eac0 contains 1 records (MAC record: 1 RLOC and 0 RAR, TTL = 1440).
005335: Dec  1 06:59:04.869: [XTR] LISP-0: WLC entry IID 8191 prefix 6cb2.ae5a.eac0/48 AP, Scheduled consumer update.
005336: Dec  1 06:59:04.869: [XTR] LISP-0: WLC client entry IID 8191 prefix 6cb2.ae5a.eac0/48 MS 192.2.99.65 SVC_VLAN_IAF_MAC type=AP rloc_addr=UNSPEC length=0, Created.
005337: Dec  1 06:59:04.869: [XTR] LISP-0: WLC client entry IID 8191 prefix 6cb2.ae5a.eac0/48 MS 192.2.99.65 SVC_VLAN_IAF_MAC type=AP rloc_addr=192.2.99.68 length=34, Updated.

*snip*
```

The edge can now use this meta data to form the VXLAN tunnel to the AP:

```
Edge2#show access-tunnel summary 

Access Tunnels General Statistics:
  Number of AccessTunnel Data Tunnels       = 1  


Name   SrcIP           SrcPort DestIP          DstPort VrfId 
------ --------------- ------- --------------- ------- ----
Ac0    192.2.99.68     N/A     192.2.12.18     4789    0   


Name   IfId            Uptime       
------ ---------- --------------------
Ac0    0x0000003D 0 days, 01:43:04
```

This concludes part II of this series. In the next part, we'll take a look at how APs are provisioned in the fabric (via DNAC), what is pushed during this provisioning and why this is a very important part of wireless integration in SDA.