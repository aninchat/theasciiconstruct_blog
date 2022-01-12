---
title: "Cisco SDA and Security Part II - micro-segmentation in SDA"
date: 2022-01-12T08:14:02+05:30
draft: false
tags: [sda, ise, sgt, cts]
description:
---

## Introduction and topology

Now that we've covered macro-segmentation in SD-Access, let's take a look at micro-segmentation. The goal of this post is to demystify the following aspects of segmentation in SDA:

* What is micro-segmentation?
* How/where are SGTs and policies for SGTs (SGACLs) created in SDA?
* Endpoint authentication/authorization - we will take a look at one endpoint to understand the basic flow behind authentication/authorization.
* How are SGACLs really pushed to the NADs? This is a common area of confusion - we will look at a flow to understand exactly WHEN the authentication server (ISE) pushes a SGACL to a NAD.
* How are SGTs added to a packet in SDA? We will breakdown an encapsulated packet and understand where the SGT value is added and why that choice was made.
* Finally, we'll take a look at an end to end flow between two hosts and understand WHERE the SGACL gets applied and how this controls communication between two endpoints. 

What we are NOT going to cover in this post:

* What an SGT actually is, what does it look like and so on. Some basic knowledge of Cisco Trustsec is assumed. 
* How NADs are added to ISE, particularly in a SDA solution where this process is automated. 
* Anyconnect NAM installation on the endpoints.
* ISE policy creation details - this post will not go into details of how to create authentication/authorization policies.
* Whitelist vs Blacklist SGT models

For the purposes of this post, we will continue to use the same topology as before:

![static1](/images/cisco/sda_security_2/security2_1.jpg)

## What is micro-segmentation? 

Within the scope of SD-Access, micro-segmentation is a methodology of controlling communication between groups of users with the help of Scalable Group Tags, commonly abbreviated to SGTs (previously known as Security Group Tags). 

Traditionally, ACLs have been leveraged for similar functionality, however, ACLs do not scale very well. Through the logic of SGTs, you can assign a value (or tag) to a group of users (users that can be grouped together based on whatever logic is relevant to you as a business) and then apply policies that control conversation between these tags (known as SGACLs). 

This methodology scales very well because users are grouped together and are assigned the same tag and thus subjected to the same policy. You no longer need to create ACLs per subnet or per user - you can create ACLs based on these tags directly. For example, an Enterprise network may have major groups of users such as IT admins, Corporate Users, Outsourced Users, Offshore Development Center (ODC) users and so on. Policies can then be created  based on tags assigned to these major groups to control communication between them.

## How are SGTs and policies created in SD-Access

Assuming your DNAC is on 1.3.1 and above, everything is centralized on DNAC and your ISE is read-only. This setting can of course be changed and ISE can be used to make changes as well (DNAC is read-only in this case). 

The "Policy" tab is where all this magic happens. Please note - all of my data is from a DNAC running 1.3.3. When you click on "Policy", this is what you see:

![static1](/images/cisco/sda_security_2/security2_2.jpg)

The "GBAC Configuration" is what allows you to set where control is for your policies:

![static1](/images/cisco/sda_security_2/security2_3.jpg)

Currently, I want to keep all control on DNAC so ISE is read-only. If you hover over the 'Group-Based Access Control" option, you should see three major things (as seen in the following image):

* Scalable Groups - this is where you create SGTs. They are automatically synced to ISE as well through ERS calls. 
* Access Contracts - contracts dictate what rules are applied between SGTs. Several pre-defined contracts exist - permit IP, deny IP and so on. 
* Policies - policies tie in SGTs with access-contracts and control communication between SGTs. 

![static1](/images/cisco/sda_security_2/security2_4.jpg)

SGTs can be created as follows, with some basic parameters that are required. For example, I am creating a SGT below called "Test_SGT" with a value of 22 and it is assigned to an existing VN,"Corp_VN":

![static1](/images/cisco/sda_security_2/security2_5.jpg)


Click on "Save". Once saved, it will sync to ISE. After a short while, you should see this SGT on ISE as well (Work Centers -> TrustSec -> Components -> Security Groups). The newly created SGT is third from bottom in this list. 

![static1](/images/cisco/sda_security_2/security2_6.jpg)

The following SGTs have already been created for the purposes of this post - Corp_Admin, Corp_Emp, ODC_Users and Guest. 

Let's say I have the following use case now - I want to ensure that traffic between ODC_Users and Corp_Emp is blocked. Let's create a policy for this. This is done in the "Policies" tab of GBAC. The flow is simple - select a source SGT -> select a destination SGT -> select contract -> Save.

![static1](/images/cisco/sda_security_2/security2_7.jpg)

![static1](/images/cisco/sda_security_2/security2_8.jpg)

![static1](/images/cisco/sda_security_2/security2_9.jpg)

![static1](/images/cisco/sda_security_2/security2_10.jpg)

The policy matrix should now have a red block indicating a deny entry for ODC_Users to Corp_Emp. 

![static1](/images/cisco/sda_security_2/security2_11.jpg)

The ISE policy matrix should have been automatically updated as well:

![static1](/images/cisco/sda_security_2/security2_12.jpg)

## Endpoint authentication/authorization

Let's take a look at one user being authenticated in this network so that we're all familiar with the general flow. My ISE authentication and authorization policies are already setup for this. The goal of my policy is to return a VLAN name and a SGT which is the most common scenario within the scope of SDA  - since this is a lab, my policies are very rudimentary. Needless to say, the authentication/authorization checks will be much more sophisticated than this in real production networks. 

The wholeidea behind taking a look at this is to understand how an authorization policy returns a VLAN and a SGT. 

So, I have a major policy-set for wired authentication (for both 802.1X and MAB) called Lab_Wired:

![static1](/images/cisco/sda_security_2/security2_13.jpg)

Any 802.1X or MAB request will hit this  policy-set. How do we differentiate between Wired and Wireless requests? The profile associated with your NADs make this very easy. For example, the default Cisco profile is associated to my NADs and this profile says the following (the profiles can be found under Administration -> Network Device Profiles):

![static1](/images/cisco/sda_security_2/security2_14.jpg)

A combination of NAS-Port-Type with the Service-Type can uniquely identify a Wired vs a Wireless request as seen above. Inside my Lab_Wired policy set, I'm simply matching on the username (which is statically set in my NAM profile on the endpoint) and the fact that it is wired 802.1X:

![static1](/images/cisco/sda_security_2/security2_15.jpg)

We hit the last rule - "aninchat_lab_host2". To authenticate the user, I am using a manually created Identity Source Sequence that looks at internal endpoints, AD etc. The user is created manually as well and is enabled, which is why it gets successfully authenticated. 

The authorization policies are then consulted and we hit the "aninchat_ODC_users"rule here.

![static1](/images/cisco/sda_security_2/security2_16.jpg)

This rule returns a result called "aninchat_test_authz" and a SGT called "ODC_Users". The result is the  following - it is simply an Access-Accept message along with a VLAN called "ODC_Users". 

![static1](/images/cisco/sda_security_2/security2_17.jpg)

The SGT is the following - it has a value of 18 (decimal) or 12 (hex) assigned to it:

![static1](/images/cisco/sda_security_2/security2_18.jpg)

A packet capture of the final Access-Accept is shown below:

![static1](/images/cisco/sda_security_2/security2_19.jpg)

The VLAN information is encoded as AV pairs. The tunnel type is VLAN and the actual VLAN name is stored in the value of Tunnel-Private-Group-Id. Finally, the SGT is under a Cisco specific AV pair and stored in the attribute called "cts:security-group-tag".

The authentication details of a session can be verified using the following command:

```
Edge2#show authentication sessions interface gig1/0/6 details 
            Interface:  GigabitEthernet1/0/6
               IIF-ID:  0x16F3E86F
          MAC Address:  000c.29f2.c674
                        fe80::4cea:ee0:9e2d:b08
         IPv4 Address:  192.2.51.2
            User-Name:  aninchat
          Device-type:  Microsoft-Workstation
          Device-name:  MSFT 5.0
               Status:  Authorized
               Domain:  DATA
       Oper host mode:  multi-auth
     Oper control dir:  both
      Session timeout:  N/A
  Acct update timeout:  172800s (local), Remaining: 10724s
    Common Session ID:  476502C00000016AB3B2E086
      Acct Session ID:  0x00000017
               Handle:  0x76000016
       Current Policy:  PMAP_DefaultWiredDot1xClosedAuth_1X_MAB


Local Policies:

Server Policies:
           Vlan Group:  Vlan: 1030
            SGT Value:  18
```

Additionally, all SGT mappings can be viewed via the following command (remember, SGTs are VRF aware as well):

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

The entire authentication/authorization workflow can be summarized as follows:

![static1](/images/cisco/sda_security_2/security2_20.jpg)