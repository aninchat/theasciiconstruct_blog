---
title: "SDA and Wireless Part I - integrating a 9800-CL WLC into SDA"
date: 2022-01-13
draft: false
tags: [sda, wireless]
description: In this post, we look at how a 9800-CL WLC is integrated into a SD-Access fabric.
---
In this post, we look at how a 9800-CL WLC is integrated into a SD-Access fabric.
<!--more-->

## Introduction and topology

This section takes you through the steps of adding a 9800-CL WLC in DNAC all the way from its discovery to provisioning. We'll be using the following topology for this:

![static1](/images/cisco/sda_wireless_1/wireless1_1.jpg)

## WLC discovery

Before a 9800-CL WLC can be added to the fabric, it must first be discovered by DNAC. In this case, we’re using an IP range to discover the WLC. Run a discovery job on the DNAC GUI (Settings -> Discovery) by filling out the following detail details:

* The discovery profile name can be anything of your liking.
* Select the ‘IP address/Range’ option and give the IP range for the WLCs.
* Preferred Management IP should be none as the WLC doesn’t have a loopback address at this point in time.
* Add credentials:
  * CLI – create a CLI set in the Design page, and input the username/password. Select this as the CLI you want to use when discovering the WLCs.
  * SNMPv2c – create a SNMPv2 read/write set in the Design page (you need to fill in the read and write community strings as configured on 9800-CL WLC. Remember each read and write window needs to be saved separately) and select the same here for your discovery.
  * HTTP is optional
  * NETCONF - netconf with a port of 830 needs to be enabled.

An example discovery is shown below:

![static1](/images/cisco/sda_wireless_1/wireless1_2.jpg)
![static1](/images/cisco/sda_wireless_1/wireless1_3.jpg)

Start the discovery by clicking the ‘Discover’ button in the bottom right corner as seen above. It should take less than 30 secs for WLCs to be discovered. Once the discovery has finished, the status of all the discovery sets (ICMP, SNMP, CLI, HTTP, if configured, and NETCONF should be green).

![static1](/images/cisco/sda_wireless_1/wireless1_4.jpg)

Rediscover the WLC if you forgot the netconf port initially (which we did and that is why you do not see netconf as green in the above discovery result) and ensure the netconf port is configured for your discovery:

![static1](/images/cisco/sda_wireless_1/wireless1_5.jpg)

Once the WLC has been discovered, it will be listed in the inventory of the DNAC GUI; the WLCs should be in a ‘reachable’ state.


**Quick Tip:**

If the WLC is in a PCF (partial collection failure) state, please verify the following:

* Check the status of all discovery data sets that were used – can you ping the WLC from DNAC? Can you SSH into the WLC using the same credentials from DNAC? Are the SNMP community strings correctly configured? And so on.
* Ensure that netconf is enabled on port 830 (‘show run all | section yang)’.
* Ensure that NTP is enabled on the WLC and is synchronized.

## WLC site assignment and HA

Now that the WLCs are in the inventory of the DNAC, you can assign it to a site and configure high availability (assuming two WLCs were discovered and are setup for HA to be configured).

**Note:** Assigning both the controllers to the same site is a pre-requisite for configuring HA.

To assign a device to a site, go to Provision -> select the check box for both WLCs -> Actions -> Provision -> Assign Device to Site -> Select the building to which WLCs should be assigned.

A WLC can only be assigned to a building or floor (something that you can assign coordinates to). APs from other buildings or floors can be mapped to this WLC while provisioning.

For configuring HA, click on the first WLC in inventory and then click again on the ‘High Availability’ tab. Once here, fill out the below information:

* The primary and secondary interfaces correspond to the primary and secondary WLCs for HA and should be selected as Gig3 for the 9800s.
* Redundancy and peer redundancy management IPs should be in same subnet as the original management IPs.
* ‘Netmask’ refers to subnet mask here and needs to be entered as a number.

![static1](/images/cisco/sda_wireless_1/wireless1_6.jpg)

**Quick Tip:**

If Gig3 is not visible in the dropdown, verify if the interface is up in the WLC. Check both the WLC CLI and the network adapters in the VM. The interface configuration should be as below which is the default configuration:

```
interface GigabitEthernet3
 negotiation auto
 no mop enabled
 no mop sysid
```

## WLC provisioning

When this process completes, only one WLC will be seen in your inventory and this can now be provisioned. During provisioning, you can assign the AP locations that this WLC will manage:

![static1](/images/cisco/sda_wireless_1/wireless1_7.jpg)

Under Advanced Configuration, a manual template can be mapped which is mainly used for pushing configurations to the WLC which are not directly available through DNAC GUI.

![static1](/images/cisco/sda_wireless_1/wireless1_8.jpg)

After this, verify the configurations and click Deploy:

![static1](/images/cisco/sda_wireless_1/wireless1_9.jpg)

Once provisioned, the provision status should change to ‘Success’ as shown below. Also, it should now appear in the fabric infrastructure page:

![static1](/images/cisco/sda_wireless_1/wireless1_10.jpg)

**Note:** It is important to assign your WLCs to a top-level hierarchy, such as a building. This allows for it to manage all floors within that building and other floors within other buildings as well.

## Adding a WLC to the fabric

You can now click on the WLC and choose ‘Add to Fabric’. This configures the WLC to be a part of the fabric while at the same time pushing appropriate configurations on the CPs (Border1 and Border2, in this case) to talk to the WLC.

A snippet of what is pushed on Border1 (as an example) is shown below:

```
*snip*

*Nov 19 07:16:04.807: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:!exec: enable
*Nov 19 07:16:06.419: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:router lisp 
*Nov 19 07:16:06.463: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ipv4 source-locator Loopback0 
*Nov 19 07:16:06.487: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:locator-set WLC
*Nov 19 07:16:06.503: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:172.21.1.3
*Nov 19 07:16:06.533: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit-locator-set 
*Nov 19 07:16:06.552: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ipv4 
*Nov 19 07:16:06.599: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:etr map-server 192.2.99.65 key uci
*Nov 19 07:16:06.641: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:etr map-server 192.2.99.69 key uci
*Nov 19 07:16:06.679: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ethernet 
*Nov 19 07:16:06.725: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:etr map-server 192.2.99.65 key uci
*Nov 19 07:16:06.773: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:etr map-server 192.2.99.69 key uci
*Nov 19 07:16:06.808: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:instance-id 4097
*Nov 19 07:16:06.828: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ipv4 
*Nov 19 07:16:06.874: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:route-export site-registrations 
*Nov 19 07:16:06.917: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:distance site-registrations 250
*Nov 19 07:16:06.955: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:map-cache site-registration 
*Nov 19 07:16:06.981: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:instance-id 4098
*Nov 19 07:16:07.005: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ipv4 
*Nov 19 07:16:07.048: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:route-export site-registrations 
*Nov 19 07:16:07.093: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:distance site-registrations 250
*Nov 19 07:16:07.137: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:map-cache site-registration 
*Nov 19 07:16:07.165: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:instance-id 4099
*Nov 19 07:16:07.189: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ipv4 
*Nov 19 07:16:07.232: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:route-export site-registrations 
*Nov 19 07:16:07.269: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:distance site-registrations 250
*Nov 19 07:16:07.308: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:map-cache site-registration 
*Nov 19 07:16:07.353: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:map-server session passive-open WLC
*Nov 19 07:16:07.371: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:site site_uci 
*Nov 19 07:16:07.402: %PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:authentication-key uci

*snip*
```

After the WLC has been added to the fabric, it should move to a ‘blue’ color within the fabric infrastructure topology , tagged with a ‘W’ within a blue circle:

![static1](/images/cisco/sda_wireless_1/wireless1_11.jpg)

This implies that it was added to the fabric successfully and is a fabric-enabled WLC.

Now, the WLC should have information regarding the CPs of the fabric and it should have an active connection to them. A simple, yet very useful command to quickly check the fabric status from the perspective of the WLC is ‘show wireless fabric summary’:

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


*snip*
```

At this point, our WLC is integrated with the fabric.