---
title: "Cisco SDA case study #1 - the case of the traffic that won't get denied"
date: 2022-01-10T19:12:48+05:30
draft: false
tags: [sda, ise, aaa, coa]
description: In this post, we will look at a peculiar case of a SGACL not denying traffic between two hosts in a SD-Access fabric.
---

## Introduction and topology

The topology in use is as follows:

![static1](/images/cisco/sda_case_study_1/case1_1.jpg)

The DNAC version we're using here is 1.3.3.3 and the ISE version is 2.4 patch 11.

The task is simple - Host2, an ODC user, should not be able to talk to Host4, a corporate user. Since they are in the same VN (Corp_VN), the goal is to achieve this segmentation via SGTs. An added requirement is that we must log this as well.


## Configuration and verification

As part of the authorization policies for these hosts, a VLAN and SGT is returned. Let's confirm the policies again. 

![static1](/images/cisco/sda_case_study_1/case1_2.jpg)


For Corporate users, we hit the authorization policy called 'aninchat_corp_users', which returns a result called 'aninchat_corp_users' and an SGT called 'Corp_Emp'. The result is the following:

![static1](/images/cisco/sda_case_study_1/case1_3.jpg)


For ODC users, we hit the authorization policy called 'aninchat_ODC_users', which returns a result called 'aninchat_test_authz' and an SGT called 'ODC_Users'. The result is the following:

![static1](/images/cisco/sda_case_study_1/case1_4.jpg)


Yes, the authorization policies are very crude and rudimentary, but that's not the goal here. The goal is to simply understand what they are doing - which is returning a VLAN and an SGT in both cases. 


Let's now confirm both hosts have been authenticated and assigned the correct VLAN and SGT. 

```
// Host4
     
Edge1#show authentication sessions interface gig1/0/22 detail
            Interface:  GigabitEthernet1/0/22
               IIF-ID:  0x1EAD82C3
          MAC Address:  000c.29c4.f1ab
                        fe80::b855:536f:7c3c:3343
         IPv4 Address:  192.2.31.3
            User-Name:  syn_gates
          Device-type:  Microsoft-Workstation
               Status:  Authorized
               Domain:  DATA
       Oper host mode:  multi-auth
     Oper control dir:  both
      Session timeout:  N/A
    Common Session ID:  4D6502C000000079E4816AAE
      Acct Session ID:  0x00000050
               Handle:  0xe000006f
       Current Policy:  PMAP_DefaultWiredDot1xClosedAuth_1X_MAB


Local Policies:

Server Policies:
           Vlan Group:  Vlan: 1029
            SGT Value:  20


Method status list:
       Method           State
        dot1x           Authc Success    

// Host2

Edge2#show authentication sessions interface gig1/0/6 details 
            Interface:  GigabitEthernet1/0/6
               IIF-ID:  0x191161C5
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
  Acct update timeout:  172800s (local), Remaining: 96036s
    Common Session ID:  476502C000000170D0C42566
      Acct Session ID:  0x0000001c
               Handle:  0x2100001c
       Current Policy:  PMAP_DefaultWiredDot1xClosedAuth_1X_MAB


Local Policies:

Server Policies:
           Vlan Group:  Vlan: 1030
            SGT Value:  18

          
Method status list:
       Method           State
        dot1x           Authc Success
```


 

Perfect - Host4 was assigned a VLAN of 1029 and an SGT of 18 (and it successfully pulled an IP address of 192.2.31.3 from the DHCP server) while Host2 was given a VLAN of 1030 and an SGT of 20 (and it successfully pulled an IP address of 192.2.51.2 from the DHCP server as well). 


Let's complete our topology with this new information.

![static1](/images/cisco/sda_case_study_1/case1_5.jpg)


As a last step, let's go and deploy a SGACL rule that denies conversation between SGT 20 to SGT 18. I've gone through the steps of configuring the policy (notice that I've chosen a contract of Deny IP Log since the intention was to log hits against the policy as well):

![static1](/images/cisco/sda_case_study_1/case1_6.jpg)

 

Save it and then deploy it.

![static1](/images/cisco/sda_case_study_1/case1_7.jpg)


At this point, a ping from Host2 to Host4 should fail. 

```
C:\Users\Host2>ping 192.2.31.3

Pinging 192.2.31.3 with 32 bytes of data:
Reply from 192.2.31.3: bytes=32 time=2ms TTL=124
Reply from 192.2.31.3: bytes=32 time<1ms TTL=124
Reply from 192.2.31.3: bytes=32 time<1ms TTL=124
Reply from 192.2.31.3: bytes=32 time<1ms TTL=124

Ping statistics for 192.2.31.3:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 2ms, Average = 0ms
```


Well. That's surprising! So, how do we troubleshoot this?


## Troubleshooting why the SGACL is not working

Since the intention was to block this conversation using SGTs, that is what we should focus on to troubleshoot this. Now, what we know so far is that the SGTs were indeed pushed correctly for each host (we verified that during the previous section). 


What about the actual SGACL though? Let's go to Edge1 to make sure that the SGACL was pushed and is applied. Remember, the SGACLs gets pushed to the NAD that has the destination SGT only which is why this will not get pushed to Edge2. 

```
Edge1#show cts role-based permissions 
IPv4 Role-based permissions default:
        Permit IP-00
IPv4 Role-based permissions from group 18:ODC_Users to group 20:Corp_Emp:
        Deny_IP_Log-01
RBACL Monitor All for Dynamic Policies : FALSE
RBACL Monitor All for Configured Policies : FALSE
```


Good - the SGACL is present. How about the contents of the ACL? 

```
Edge1#show ip access-lists Deny_IP_Log-01
Role-based IP access list Deny_IP_Log-01 (downloaded)

** no output **
```


Okay, that's definitely odd. The ACL exists but it is empty - there are no ACEs inside it. Let's deploy the policy again and capture the packet exchange between the NAD (Edge1, in this case) and the ISE to understand whats happening. 


To re-deploy, I simply revert the change and re-configure the policy, and deploy again. A packet capture was already setup prior to this. Now, this is the flow I see (this is also the generic flow you would typically see when a SGACL is pushed/deployed):

![static1](/images/cisco/sda_case_study_1/case1_8.jpg)


The important things to notice here - ISE informs ALL NADs when a policy needs to be deployed by sending a CoA with an AV-pair that includes the destination SGT that was in the policy. 


Any NAD that has downloaded this SGT will then send an Access-Request to ISE to download the actual policy. ISE then returns the SGACL name, following which the NAD again sends another Access-Request requesting for the contents of this ACL. ISE then replies with an Access-Accept which includes the ACEs inside this ACL. 


Look at the last Access-Accept in the above flow (where ISE should have sent the ACEs) - it is empty! ISE returned nothing for some reason. Our packet capture also confirms this entire flow. 


ISE sending a CoA to the NAD:

![static1](/images/cisco/sda_case_study_1/case1_9.jpg)


Edge1 sending an Access-Request for this SGT:

![static1](/images/cisco/sda_case_study_1/case1_10.jpg)


ISE sending the SGACL name back:

![static1](/images/cisco/sda_case_study_1/case1_11.jpg)


Edge1 sending another Access-Request with the ACL name to get its contents from ISE:

![static1](/images/cisco/sda_case_study_1/case1_12.jpg)


ISE responding back with an empty ACE list:

![static1](/images/cisco/sda_case_study_1/case1_13.jpg)


This is why the ACL is empty - its because ISE never sent anything back! Why would ISE do this? 


Remember, by default, on 2.4 patch 11, there are 4 contracts available:

1. Permit IP
2. Deny IP
3. Permit IP Log
4. Deny IP Log


The "Log" contracts allow for logging of SGACL hits - that's the only difference between them and the regular contracts.  These ACLs are maintained on ISE itself - the logical next step would be to check the validity of these ACLs on ISE. 


Small hiccup here though - the default contracts CANNOT be viewed in the GUI. Well, good thing we have ERS enabled already! I can use some simple GET calls and leverage some ISE APIs to try and pull these contracts. 

```
ANINCHAT-M-R0U2:Downloads aninchat$ curl -k -X GET --include --header 'Accept: application/json' --user admin:C1sc0123  https://10.104.152.120:9060/ers/config/sgacl/
HTTP/1.1 200 OK
Cache-Control: no-cache, no-store, must-revalidate
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: JSESSIONIDSSO=9EE22DDCA5A7184197A4CBA5632AA273; Path=/; Secure; HttpOnly
Set-Cookie: APPSESSIONID=7C9B8FAF7F14815C925A35C54DF91B85; Path=/ers; Secure; HttpOnly
Pragma: no-cache
Date: Sun, 10 May 2020 12:47:34 GMT
Content-Type: application/json;charset=utf-8
Content-Length: 1336
Server:  

{
  "SearchResult" : {
    "total" : 4,
    "resources" : [ {
      "id" : "92919850-8c01-11e6-996c-525400b48521",
      "name" : "Deny IP",
      "description" : "Deny IP SGACL",
      "link" : {
        "rel" : "self",
        "href" : "https://10.104.152.120:9060/ers/config/sgacl/92919850-8c01-11e6-996c-525400b48521",
        "type" : "application/json"
      }
    }, {
      "id" : "995DD09221044680E053B10F380AF59D",
      "name" : "Deny_IP_Log",
      "description" : "Deny IP with logging",
      "link" : {
        "rel" : "self",
        "href" : "https://10.104.152.120:9060/ers/config/sgacl/995DD09221044680E053B10F380AF59D",
        "type" : "application/json"
      }
    }, {
      "id" : "92951ac0-8c01-11e6-996c-525400b48521",
      "name" : "Permit IP",
      "description" : "Permit IP SGACL",
      "link" : {
        "rel" : "self",
        "href" : "https://10.104.152.120:9060/ers/config/sgacl/92951ac0-8c01-11e6-996c-525400b48521",
        "type" : "application/json"
      }
    }, {
      "id" : "995DD092210C4680E053B10F380AF59D",
      "name" : "Permit_IP_Log",
      "description" : "Permit IP with logging",
      "link" : {
        "rel" : "self",
        "href" : "https://10.104.152.120:9060/ers/config/sgacl/995DD092210C4680E053B10F380AF59D",
        "type" : "application/json"
      }
    } ]
  }
  ```


As you can see, each of these contracts are referenced via an ID. To look at the actual contents of the ACL, you can reference this ID in your GET call. 

```
// GET call for Deny IP contract

ANINCHAT-M-R0U2:Downloads aninchat$ curl -k -X GET --include --header 'Accept: application/json' --user admin:C1sc0123  https://10.104.152.120:9060/ers/config/sgacl/92919850-8c01-11e6-996c-525400b48521
HTTP/1.1 200 OK
Cache-Control: no-cache, no-store, must-revalidate
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: JSESSIONIDSSO=086BFA3B5D0CBA6C3242643D971AEEA2; Path=/; Secure; HttpOnly
Set-Cookie: APPSESSIONID=D124269656F0179F4CE49242A0EFC0C8; Path=/ers; Secure; HttpOnly
Pragma: no-cache
ETag: "7A78BE2E98DD06885AC1982886EAC90C"
Date: Sun, 10 May 2020 12:51:17 GMT
Content-Type: application/json;charset=utf-8
Content-Length: 366
Server:  

{
  "Sgacl" : {
    "id" : "92919850-8c01-11e6-996c-525400b48521",
    "name" : "Deny IP",
    "description" : "Deny IP SGACL",
    "generationId" : "0",
    "aclcontent" : "deny ip",
    "link" : {
      "rel" : "self",
      "href" : "https://10.104.152.120:9060/ers/config/sgacl/92919850-8c01-11e6-996c-525400b48521",
      "type" : "application/json"
    }
  }
}

// GET call for Deny IP Log contract

ANINCHAT-M-R0U2:Downloads aninchat$ curl -k -X GET --include --header 'Accept: application/json' --user admin:C1sc0123  https://10.104.152.120:9060/ers/config/sgacl/995DD09221044680E053B10F380AF59D
HTTP/1.1 200 OK
Cache-Control: no-cache, no-store, must-revalidate
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: JSESSIONIDSSO=ADE02E0521F31D1F15C0393019B01161; Path=/; Secure; HttpOnly
Set-Cookie: APPSESSIONID=37B52F4E5310562DA4583FC5EE787E84; Path=/ers; Secure; HttpOnly
Pragma: no-cache
ETag: "FF7C1B3AC56640A5FD090C726154E86C"
Date: Sun, 10 May 2020 12:51:27 GMT
Content-Type: application/json;charset=utf-8
Content-Length: 362
Server:  

{
  "Sgacl" : {
    "id" : "995DD09221044680E053B10F380AF59D",
    "name" : "Deny_IP_Log",
    "description" : "Deny IP with logging",
    "generationId" : "1",
    "aclcontent" : "",
    "link" : {
      "rel" : "self",
      "href" : "https://10.104.152.120:9060/ers/config/sgacl/995DD09221044680E053B10F380AF59D",
      "type" : "application/json"
    }
  }
}
```


The first call pulls the content of the Deny IP contract - focus on the "aclcontent" part. It correctly contains a "deny ip" ACE. 


The next call pulls the content of the Deny IP Log contract - again, focus on the "aclcontent" part. It is empty! 


And finally, we've come to the source of our problem - the log contracts are empty on ISE. Because of this, when you reference these contracts, ISE does not return any ACE at all. 


Naturally, from a troubleshooting perspective, there is where we stop - this was obviously a bug in ISE which is fixed in later patches.
