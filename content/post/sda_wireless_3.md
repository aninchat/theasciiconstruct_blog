---
title: "SDA and Wireless Part III - understanding AP configurations on the 9800"
date: 2022-01-13T13:52:53+05:30
draft: false
tags: [sda, wireless]
description: In this post, we look at the various access point specific configurations that are pushed to a fabric-enabled WLC.
---

## Introduction

The configurations pushed to a 9800 WLC (for an AP) can be a little convoluted. This post aims at demystifying that. 

At present, the following APs have registered with this WLC:

```
9800-WLC2#show ap summary 
Number of APs: 2

AP Name                            Slots    AP Model  Ethernet MAC    Radio MAC       Location              Country     IP Address                                 State         
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
AP380E.4D5B.0D40                     3      3802I     380e.4d5b.0d40  6cb2.ae5a.eac0  default location      J4          192.2.12.18                                Registered    
AP7872.5DAB.D2E6                     3      3802I     7872.5dab.d2e6  7872.5dbd.6ca0  default location      J4          192.2.12.21                                Registered
```

Let’s consider an AP that has been provisioned already; the following configuration is present under the AP:

```
9800-WLC2#show run | sec 380E.4D5B.0D40
ap 380E.4D5B.0D40
 policy-tag Test2_aninchat
 rf-tag TYPICAL
 site-tag default-site-tag-fabric
```

Three major things are pushed for the AP:
* A policy-tag 
* A rf-tag
* A site-tag 

We’ll take a look at each of these individually.

## Policy tags

A policy-tag essentially ties together a WLAN and a wireless profile policy. As we can see from the above, we have a policy-tag called ‘Test2_aninchat’ called under the AP. This is configured as follows:

```
9800-WLC2(config)#wireless tag policy <name of the policy tag>
9800-WLC2(config-policy-tag)#wlan <WLAN_name> policy <wireless profile policy name>
```

You can verify this using the following command:

```
9800-WLC2#show wireless tag policy detailed Test2_aninchat
Policy Tag Name : Test2_aninchat
Description     : 

Number of WLAN-POLICY maps: 1
WLAN Profile Name                 Policy Name                             
------------------------------------------------------------------------
Test2_anin_Global_F_0922d0c8      Test2_anin_Global_F_0922d0c8
```

The WLAN configuration is where you define the SSID, the kind of authentication you want to do for that SSID and so on. This can be confirmed as follows:

```
9800-WLC2#show run | sec wlan Test2_anin_Global_F_0922d0c8
wlan Test2_anin_Global_F_0922d0c8 18 Test2_aninchat
 no ccx aironet-iesupport
 security wpa psk set-key ascii 0 C1sc0123
 no security wpa akm dot1x
 security wpa akm psk
 no shutdown
```

As you can see, we’re doing a simple pre-shared key authentication with a password of ‘C1sc0123’ and the SSID is called Test2_aninchat. To quickly look at all WLANs and their associated SSIDs, you can use the command ‘show wlan summary’:

```
9800-WLC2#show wlan summary 

Number of WLANs: 2

WLAN Profile Name                     SSID                             Status 
-----------------------------------------------------------------------------
17   Test_aninc_Global_F_dee1ad6f     Test_aninchat                    UP     
18   Test2_anin_Global_F_0922d0c8     Test2_aninchat                   UP
```

The wireless profile policy is where you define AAA parameters, QoS and netflow policies, an IP pool (from a fabric perspective, a L2 VNID) and so on. This can be confirmed using the following:

```
9800-WLC2#show run | sec wireless profile policy Test2_anin_Global_F_0922d0c8
wireless profile policy Test2_anin_Global_F_0922d0c8
 aaa-override
 no central dhcp
 no central switching
 description Test2_anin_Global_F_0922d0c8
 dhcp-tlv-caching
 fabric Test2_anin_Global_F_0922d0c8
 http-tlv-caching
 ipv4 flow monitor wireless-avc-basic input
 ipv4 flow monitor wireless-avc-basic output
 service-policy input test2
 service-policy output test2
 no shutdown 
```

The wireless profile for fabric will tell you what L2VNID was mapped to this wireless profile policy:

```
9800-WLC2#show run | sec wireless profile fabric Test2_anin_Global_F_0922d0c8
wireless profile fabric Test2_anin_Global_F_0922d0c8
 client-l2-vnid 8196
 description Test2_anin_Global_F_0922d0c8
```

Thus, at a high level, both the WLAN and wireless profile policy feed into the policy-tag:

![static1](/images/cisco/sda_wireless_3/wireless3_1.jpg)

## Site tags

The site-tag is used to determine the AP profile that will be used for the APs. A quick summary of all site-tags can be seen using the following command:

```
9800-WLC2#show wireless tag site summary 

Number of Site Tags: 2

Site Tag Name                     Description                             
------------------------------------------------------------------------
default-site-tag                  default site tag                        
default-site-tag-fabric           defaultFabricSiteTagDesc
```

We are using ‘default-site-tag-fabric’ for the APs, so let’s look at that in detail (this site tag is created automatically by DNAC once the WLC is provisioned to join the fabric):

```
9800-WLC2#show wireless tag site detailed default-site-tag-fabric

Site Tag Name        : default-site-tag-fabric
Description          : defaultFabricSiteTagDesc
----------------------------------------  
AP Profile           : default-ap-profile-fabric
Local-site           : Yes
Image Download Profile: default-me-image-download-profile
```

As you can see, an AP profile is called within the site-tag. The AP profile can be confirmed as below:

```
9800-WLC2#show run | sec ap profile default-ap-profile-fabric
ap profile default-ap-profile-fabric
 hyperlocation ble-beacon 0
 hyperlocation ble-beacon 1
 hyperlocation ble-beacon 2
 hyperlocation ble-beacon 3
 hyperlocation ble-beacon 4
 mgmtuser username cisco password 0 cisco secret 0 cisco
 ssh
 telnet
```

Among other things, the AP profile is where you set login information for your AP as well.

## RF tags

Finally, the RF tag is where you set your RF parameters. A summary of all RF tags can be seen using the following command:

```
9800-WLC2#show wireless tag rf summary 

Number of RF Tags: 3

RF tag name                       Description                             
------------------------------------------------------------------------
test                                                                      
TYPICAL                                                                   
default-rf-tag                    default RF tag
```

We are using a RF tag of ‘TYPICAL’. You can view this in detail using:

```
9800-WLC2#show wireless tag rf detailed TYPICAL 

Tag Name             : TYPICAL
Description          : 
----------------------------------------  
5ghz RF Policy       : Typical_Client_Density_rf_5gh
2.4ghz RF Policy     : Typical_Client_Density_rf_24gh
```

## Comparing an unprovisioned and a provisioned AP

Remember, an un-provisioned AP will not have any of these configurations pushed for it on the WLC. For example, we just brought up another AP in the lab:

```
9800-WLC2#show ap summary | in 380E.4D5B.0C82 
AP380E.4D5B.0C82                     3      3802I     380e.4d5b.0c82  6cb2.ae5a.dee0  default location      J4          192.2.12.22                                Registered
```

This has not been provisioned yet:

![static1](/images/cisco/sda_wireless_3/wireless3_2.jpg)

Since this is not provisioned, the WLC has no configuration for this AP. Compare this to a provisioned AP, which has all the relevant configuration:

```
9800-WLC2#show run | sec 380E.4D5B.0C82
9800-WLC2#show run | sec 380E.4D5B.0D40
ap 380E.4D5B.0D40
 policy-tag Test2_aninchat
 rf-tag TYPICAL
 site-tag default-site-tag-fabric
```

This is why it is very important to verify if these commands were pushed correctly on AP provision. If there is an issue here, then you can see varying problems – clients not working after connecting to this AP, clients having issues roaming to this AP and so on.

## Verifying from the WLC GUI

We’ve looked at all of this information via the CLI, however, for completeness sake, you can confirm the same via the GUI as follows. Login to the GUI and Configuration -> Tags will give you an overview of all tags (policy, site and rf) created for this WLC:

![static1](/images/cisco/sda_wireless_3/wireless3_3.jpg)
![static1](/images/cisco/sda_wireless_3/wireless3_4.jpg)

As you can see, there are tabs for each of the different types of tags. From here, you can click on any of these to fetch more details around this. For example, click on the policy tag ‘Test2_aninchat’ and you get:

![static1](/images/cisco/sda_wireless_3/wireless3_5.jpg)

This shows you the same information we saw on the CLI – the WLAN and the profile policy associated to this tag. To get details about the profile policy itself, you can go to Configuration -> Policy:

![static1](/images/cisco/sda_wireless_3/wireless3_6.jpg)

This will list all policies created on the WLC:

![static1](/images/cisco/sda_wireless_3/wireless3_7.jpg)

From here, you can click on any of the profile policies to get more details around this:

![static1](/images/cisco/sda_wireless_3/wireless3_8.jpg)

Back under the configuration option, you can go to Configuration -> WLAN to see all WLANs created for the WLC:

![static1](/images/cisco/sda_wireless_3/wireless3_9.jpg)

From here, you can click on any of the WLANs to get more details around it:

![static1](/images/cisco/sda_wireless_3/wireless3_10.jpg)

The ‘Security’ tab allows you to see all the security aspects of this WLAN:

![static1](/images/cisco/sda_wireless_3/wireless3_11.jpg)

This concludes part III of the SDA and wireless series. I hope this was informative!