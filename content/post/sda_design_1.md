---
title: "Cisco SDA design study #1 - understanding user authentication in the fabric"
date: 2021-12-27T18:03:58+05:30
draft: false
tags: [sda, ise, aaa]
description: In this post, we look at how a typical user gets authenticated and authorized in a SD-Access fabric.
---
In this post, we look at how a typical user gets authenticated and authorized in a SD-Access fabric.
<!--more-->

## Introduction and topology

The premise of this design discussion is that the network is separated into several users from different organizations that come and work out of your campus. The requirement is to have these users authenticated, authorized via ISE (with AD integrated) and then once they are found to be posture compliant, they should be placed into their organizational specific VLAN. 


The topology is a typical SDA design - we have several edge switches that are uplinked to two borders. These border switches are doing a L3 handoff (via BGP) to the fusion which connects to a Palo Alto cluster as well as their shared services segment. The Palo Alto maintains a User Identity table based on which Internet access is allowed (if the username matches the IP address of the client). 

![static1](/images/cisco/sda_design_1/design_1.jpg)



The goal of this design study is to understand how the premise is translated to actual network design/configuration and the general flow of the network when the user (Host1, as an example here) connects to the edge for the first time. 


Please note - some of the captures here have been sanitized to protect actual identities and customer nomenclature. This may make it harder for you to view the packet captures or ISE flow snippets, however this is a necessary security precaution that I must take. 

## ISE policy flow

Let's begin!


First, let's take a look at the entire policy flow from the perspective of ISE when Host1 first connects to the network over a wired connection. The below image snippet has been taken from the Live Logs section of the ISE GUI:


![static1](/images/cisco/sda_design_1/design_2.jpg)

### Authentication of the user 

The first thing to notice is the identity – this is in the format of username, host/<machine name>. This implies that we’re doing both user and machine authentication at the same time using EAP chaining. 


The client is using dot1x, which is why we hit the “redacted_Wired_Monitor>>Dot1x” policy for authentication. This policy can be found under policy-sets -> authentication policy:

![static1](/images/cisco/sda_design_1/design_3.jpg)




This policy uses the “redacted_Source_Sequence” which is a user defined identity sequence under Administration -> Identity Management -> Identity Source Sequences. 

![static1](/images/cisco/sda_design_1/design_4.jpg)




This sequence checks the username against any Internal Users that may be defined in ISE and then moves on to check the AD called ‘redacted'. The result of this policy can be seen towards the right of the following image – if the authentication fails, we reject; if the user is not found, we reject. If the process fails somehow, we drop the packet. 


![static1](/images/cisco/sda_design_1/design_5.jpg)


 ### Authorization of the user 

At the same time, for authorization, the user is put into Unknown_MachinePlusUser. This can be found under policy-sets -> authorization policies:


![static1](/images/cisco/sda_design_1/design_6.jpg)


The user hits this profile because at this stage, user and machine authentication has succeeded AND the user is found in the group of domain users AND the posture status is currently unknown. This is the criteria for this authorization profile. 


  

The result of this policy is ‘Wired_Posture_Result’. This can be found under Policy -> Results:

![static1](/images/cisco/sda_design_1/design_7.jpg)
  

This policy returns an Access Accept Radius message along with a VLAN (say VLAN X) and a client provisioning portal (web redirection) for posture. Notice how we’re returning the VLAN name and not the number itself – the NAD maps the VLAN name to the number and configures it appropriately.

### Posture

The client is now able to successfully request for an IP address via DHCP. It gets an IP address within the range of this VLAN, say VLAN X.

![static1](/images/cisco/sda_design_1/design_8.jpg)

 

At this point, posture checks start via web redirection and the client provisioning portal. The actual portal itself can be found under Administration -> Device Portal Management -> Client Provisioning. The portal has a policy associated to it which can be found under Work Center -> Posture -> Client Provisioning -> Client Provisioning Policy (the policy can be found here as well):

![static1](/images/cisco/sda_design_1/design_9.jpg)



This portal does several checks including a posture compliance check. This requires a validation of the anyconnect configuration on the client. The client has an anyconnect compliance module installed that can report the posture status back to ISE. 


 ### Authorization post posture 

Once this is done and the user needs to be authorized again post posture, the authorization policy it hits is ‘Compliant_MachinePlusUser_redacted' – this is because user and machine authentication are both a success AND the user is part of the redacted AD group AND the posture status is compliant, as can be seen from the policy criteria below:

![static1](/images/cisco/sda_design_1/design_10.jpg)




The right most option (which has been redacted) is where the SGT is assigned as well. As you can see, the result of this authorization policy is 'Wired_Compliance_redacted'. This result can be viewed in detail as below:

![static1](/images/cisco/sda_design_1/design_11.jpg)



  

ISE returns an Access Accept Radius message and a new VLAN for the client. The av pairs within the Access Accept indicate that the tunnel type is VLAN, the medium is IEEE 802 and finally the tunnel private group ID defines the VLAN (or VLAN name, in this case). This is the organization specific VLAN that the client finally moves to post compliance. 

![static1](/images/cisco/sda_design_1/design_12.jpg)




Post this, ISE sends a CoA for reauthentication. The packet looks like this:

![static1](/images/cisco/sda_design_1/design_13.jpg)




Visually, this looks like so:

![static1](/images/cisco/sda_design_1/design_14.jpg)




Post COA, the DHCP process starts again, but this time it hits the new VLAN:

![static1](/images/cisco/sda_design_1/design_15.jpg)



  

Because the Palo Alto has been configured as a logging server on ISE, all accounting messages are sent to the Palo Alto. When the clients IP addresses changes, the NAD generates a Radius accounting message to ISE – ISE in turn sends this to Palo Alto so that the new IP address can be mapped to the user.

![static1](/images/cisco/sda_design_1/design_16.jpg)

  

This process can be visualized as below:

![static1](/images/cisco/sda_design_1/design_17.jpg)



  

This completes an end to end flow of a user connecting to the network, being authenticated and authorized by ISE and being allocated to the correct VLAN post compliance (assuming it was compliant).