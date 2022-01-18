---
title: "Cisco SDA and Security Part III - micro-segmentation in SDA continued"
date: 2022-01-12T08:14:04+05:30
draft: false
tags: [sda, ise, sgt, micro-segmentation]
description: In this post, we look at how SGACLs are pushed to NADs, with clear packet captures and packet walks.
---
In this post, we look at how SGACLs are pushed to NADs, with clear packet captures and packet walks. We also see how SGTs are added to the VXLAN header.
<!--more-->

## Introduction

This post is a continuation of Part II of my Cisco SDA and Security series. If you haven't read that yet, please do for some context.

## How are SGACLs really pushed to NADs?

This is an area of confusion for many and I have heard varying answers to this - the most common misconception is that it is pushed based on data traffic which is just not true. Let's try to clear this up. 

The first and foremost (and the most important) rule of SGACLs is this - it is always applied at the egress. For example, if in our case, Host2 tries to reach Host4 and we have an SGACL blocking this - this rule should be applied at the destination NAD (Edge1) and not the source NAD. This naturally implies that the SGACLs need to ONLY be downloaded to the destination NADs for a particular flow. Sounds confusing? It is. So, let's take a look at an example for some clarity. 

Remember back to when we created an SGACL that denied communication from ODC_Users (SGT value of 18 in decimal) to Corp_Emp (SGT value of 20 in decimal). Even though this SGACL is created, it is NOT pushed to any device (yet). NADs will PULL the SGACL when they meet the requirement - the requirement is that they must have an SGT assigned that matches the destination SGT in the SGACL. 


Go back to the onboarding of Host2 - as part of the authorization policy, an Access-Accept is sent back by ISE, along with an SGT and VLAN. 

![static1](/images/cisco/sda_security_3/security3_1.jpg)

The NAD (Edge2) immediately sends another Access-Request to ISE (refer to the sequencing flow in Part II - post EAP Success, an Access-Request is sent). This is the NAD telling ISE that it now knows of SGT 18. 

![static1](/images/cisco/sda_security_3/security3_2.jpg)

The two most important aspects of this Access-Request are the username (which is sent to #CTSREQUEST#) and the AV-pair which lists the SGTs that the NAD knows about (cts-rbacl-source-list). You can see in the below packet capture that it is set to 0012 which is 18 in decimal. 

![static1](/images/cisco/sda_security_3/security3_3.jpg)

ISE looks at its SGACLs and if it finds any SGACL where the destination SGT is 18, it will send this to the NAD (via AV pairs) by replying with an Access-Accept. Even if it does not find any SGACL with this destination SGT, it will send an Access-Accept back with no SGACL. 


The following packet capture confirms that in this case, since there is no matching SGACL with SGT 18 as the destination, it simply sends back an empty SGACL as part of the Access-Accept:

![static1](/images/cisco/sda_security_3/security3_4.jpg)

These #CTSREQUEST# based Access-Requests can clearly be seen in the Radius Live Logs:

![static1](/images/cisco/sda_security_3/security3_5.jpg)

The detailed report for this will also show you who originated this, what the Cisco AV-pairs are and what the result is. This matches up with what we saw in the packet captures as well. 

![static1](/images/cisco/sda_security_3/security3_6.jpg)
![static1](/images/cisco/sda_security_3/security3_7.jpg)

A final confirmation is to list the SGACLs on the NAD itself, which can be done via the following commands:

```
// SGACLs
 
Edge2#show cts role-based permissions 
IPv4 Role-based permissions default:
        Permit IP-00
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE

// counter for SGACLs

Edge2#show cts role-based counters 
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          1825522    20531610   0          0   
```

As you can see, only a default permit any any entry is present (which is always there in a blacklist SGT model). This confirms that no SGACLs were pushed to Edge2 even after Host2 was authenticated and authorized (despite the existence of an SGACL which had Host2s assigned SGT). 

Let's now onboard Host4 to show WHEN SGTs are actually pushed. Again, the authentication/authorization process is the same. As part of the authorization policy, we return an Access-Accept with a VLAN called "Corp_Users" and an SGT = Corp_Emp (a decimal value of 20).

![static1](/images/cisco/sda_security_3/security3_8.jpg)

Once Edge1 gets this SGT and maps it to Host4s IP address, it immediately sends out an Access-Request, similar to what Edge2 did when Host2 was being onboarded. The following packet capture confirms that this was a CTSREQUEST and the AV-pairs listed SGT 20.

![static1](/images/cisco/sda_security_3/security3_9.jpg)

This time, when ISE looks at its SGACLs, it determines that there is one SGACL where SGT 20 is the destination SGT. It returns this SGACL as part of the Access-Accept, as seen below:

![static1](/images/cisco/sda_security_3/security3_10.jpg)

Confirm that this SGACL is now populated on Edge1:

```
// SGT binding on Edge1
     
Edge1#show cts role-based sgt-map vrf Corp_VN all
%IPv6 protocol is not enabled in VRF Corp_VN
Active IPv4-SGT Bindings Information

IP Address              SGT     Source
============================================
192.2.31.3              20      LOCAL

IP-SGT Active Bindings Summary
============================================
Total number of LOCAL    bindings = 1
Total number of active   bindings = 1

// SGACL downloaded to Edge1

Edge1#show cts role-based permissions 
IPv4 Role-based permissions default:
        Permit IP-00
IPv4 Role-based permissions from group 18:ODC_Users to group 20:Corp_Emp:
        Deny IP-00
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE

// SGACL counters shows this new SGACL

Edge1#show cts role-based counters 
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          1829375    2458582    0          0         
18      20      0          0          0          0          0          0  
```

The entire authentication/authorization process is seen below for this host on ISE Live Logs as well:

![static1](/images/cisco/sda_security_3/security3_11.jpg)

A detailed report for the CTSREQUEST confirms this SGACL pull/push model:

![static1](/images/cisco/sda_security_3/security3_12.jpg)
![static1](/images/cisco/sda_security_3/security3_13.jpg)

## How are SGTs added to a packet in SDA? 

Okay, at this point, we've understood all of the moving pieces  behind this - all of this forms the policy plane. The actual forwarding decisions are taken in the data-plane though, so let's try and ping from Host2 to Host4 and see what happens:

```
C:\Users\admin-PC2> ping 192.2.31.3

Pinging 192.2.31.3 with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 192.2.31.3:
	Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

All pings fail. Good - this is what we expected! Let's unravel these packets to understand exactly what happened. 

So, the ICMP packet comes to Edge1. Edge1 finds out who the destination RLOC is via the LISP process, encapsulates the packet and sends it out. The VXLAN header should have the VNI (which matches the LISP instance ID for this VRF) and more importantly, it should have the SGT for the source added as well. 

![static1](/images/cisco/sda_security_3/security3_14.jpg)

We can confirm the same via a packet capture as well. This is the packet that ingresses Edge1:

![static1](/images/cisco/sda_security_3/security3_15.jpg)

The Group Policy ID gives the source SGT here. Why is the destination SGT unknown (or not populated, which essentially means unknown)? Well, does Edge2 have any idea of what the destination SGT should be? No. We can confirm this in the SGT table:

```
Edge2#show cts role-based sgt-map vrf Corp_VN all
%IPv6 protocol is not enabled in VRF Corp_VN
Active IPv4-SGT Bindings Information

IP Address              SGT     Source
============================================
192.2.51.2              18      LOCAL

IP-SGT Active Bindings Summary
============================================
Total number of LOCAL    bindings = 1
Total number of active   bindings = 1
```

So, naturally, it can only populate the source SGT at this point in time. The packet is now routed via towards the destination RLOC, Edge1. When Edge1 gets it, it will do a SGT lookup for the destination and form the complete SGT set - a source of SGT 18 and a destination of SGT 20. 


Because an SGACL exists that is denying this flow, it will get dropped. We can confirm this using the SGT counters:

```
Edge1#show cts role-based counters 
Role-based IPv4 counters
From    To      SW-Denied  HW-Denied  SW-Permitt HW-Permitt SW-Monitor HW-Monitor
*       *       0          0          258        254        0          0         
18      20      0          4          0          0          0          0  
```

Host4 had sent four ICMP requests for Host2. All of these are accounted for as drops in the SGT counter. 

This completes an end to end flow of how we can use SGTs (a form of micro-segmentation) to control communication between hosts in the same VN. 


## Why SGTs inside the VXLAN header?

Traditionally, SGTs can be propagated via inline tagging. This implies that a Cisco metadata header is added (right after the Ethernet header). This header contains the SGTs that are added to the packet. This method is commonly known as inline tagging.

So, why not simply use inline tagging? Well, as the name states, "Cisco" metadata could potentially cause a situation where a device does not understand it. When this happens, the device would drop the packet. Thus, to ensure there are no backward compatibility issues, a choice was made to add the SGTs within the VXLAN header itself since VXLAN is an IEEE standard. 

This concludes Part III of this series - it's been a long one but if you've reached the end, I hope it was well worth it.
