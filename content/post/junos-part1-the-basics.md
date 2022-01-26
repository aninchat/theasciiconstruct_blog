---
title: "Junos Part I - the basics"
date: 2022-01-24
draft: false
tags: [juniper, junos, jncia, stp]
description: "In this post, we take an introductory look at the operating system used in Juniper platforms, called Junos OS. This post will cover basic routing, switching and the most common tools you'd want to use for troubleshooting on the platform."
---
In this post, we take an introductory look at the operating system used in Juniper platforms, called Junos OS. This post will cover basic routing, switching and the most common tools you'd want to use for troubleshooting on the platform.
<!--more-->

## Introduction and topology

As a new user of the Junos OS, I thought I'd take this opportunity to blog about what I generally start with when learning a new platform/OS. It's important to understand the basics before moving onto the more complicated technologies. For me, the basics have always included L2 protocols like STP, some form of link aggregation (LAG), at least one IGP, and then finally, understand the tools available to troubleshoot a problem on the platform. Naturally, you also need to get comfortable with navigating the CLI. 

To that end, here's the topology that we'll be using for this post:

![topology](/images/juniper/junos_1/junos_basics_1.jpg)

## Navigating the CLI

### Factory default settings

The first, more natural thing, is to learn how to navigate the CLI itself. The Junos OS is a beast of it's own - at first glance, it seems far more complex than your typical run-of-the-mill Cisco style CLI. You need to give it some time - the kind of quality of life features the Junos OS has is AMAZING. 

When you first boot up a factory default Juniper switch (virtual, or hardware), you'll see something like this:

```
Amnesiac (ttyd0)

login:
```

'Amnesiac' essentially means the switch is in factory default state. At this point, you can login with 'root' and no password. 

```
Amnesiac (ttyd0)

login: root

--- JUNOS 21.4R1.12 built 2021-12-17 14:37:27 UTC

root@:RE:0%
```

### CLI modes

In general, you can be in one of three places in the OS - the shell, operational mode or configuration mode. A non-root user will land in operational mode when logging in. 

Moving from the shell to operational mode can be done using the 'cli' command, and moving from operational mode to configuration mode can be done using the 'configure' command. 

```
// shell 

root@SW1:RE:0%

// moving from shell to operational mode

root@:RE:0% cli
{master:0}

// moving from operational mode to configuration mode

root> configure 
Entering configuration mode

{master:0}[edit]
```

### Basic configuration and syntax

Junos OS configuration is hierarchical. Notice the 'edit' keyword. in brackets, when you entered configuration mode above. This signifies that you are in the edit mode, which is the top most configuration hierarchy. From here, you can start configuring things in two ways:

1. You can use set commands directly from the edit mode.
2. You can go further down the hierarchy by using additional 'edit' commands and then use 'set' commands.

To demonstrate this, let's set a root password for the device, and change the hostname. 

Using set commands, we can do this as follows:

```
{master:0}[edit]
root# set system host-name SW1 

{master:0}[edit]
root# set system root-authentication plain-text-password 
New password:
Retype new password:

{master:0}[edit]
```

Using edit commands to go down the hierarchy, we can do the following:

```
{master:0}[edit]
root# edit system 

{master:0}[edit system]
root# set host-name SW1
```

Notice how the OS tells you exactly where you are in the hierarchy. The 'edit' changed to 'edit system' because you went to the system level hierarchy. From here, you can issue system level set commands. 

To set the root password, we can do the following:

```
{master:0}[edit system]
root# set root-authentication plain-text-password 
New password:
Retype new password:

{master:0}[edit system]
root# 
```

### Saving configuration

Junos OS works with the concept of an active and candidate configuration. The active configuration is what the device booted up with and is actively running/using. A candidate configuration is created as soon as you enter configuration mode by typing 'configure' or 'edit' while in operational mode.

The candidate configuration is essentially a copy of the active configuration, but any changes you make in configuration mode are made to the candidate configuration and not the active configuration. Let that sink in - this is a big shift to what you were potentially used to with vendors like Cisco, and operating systems like IOS/IOS-XE/NX-OS. Any changes made on these would be applied immediately - the purpose of saving the configuration was just to ensure that when the device boots up again, these changes are a part of the configuration.

In Junos OS, to move the changes from the candidate configuration to active configuration (essentially making the candidate configuration the active), you need to 'commit' your changes.

```
root# commit ?
Possible completions:
  <[Enter]>            Execute this command
  activate             Activate a previously prepared commit
  and-quit             Quit configuration mode if commit succeeds
  at                   Time at which to activate configuration changes
  check                Check correctness of syntax; do not apply changes
  comment              Message to write to commit log
  confirmed            Automatically rollback if not confirmed
  no-synchronize       Don't synchronize commit
  peers-synchronize    Synchronize commit on remote peers
  prepare              Prepare for an upcoming commit activation
  scripts              Push scripts to other RE
  synchronize          Synchronize commit on both routing engines
  |                    Pipe through a command
```

As you can see, there are multiple options to commit changes. The simplest way is just to type commit, and press enter. 

```
root# commit    
configuration check succeeds
commit complete
```

The OS checks for syntax errors, and if none are found, the changes are committed. There are several other handy ways to commit, some of which I use often:

1. commit check - this simply checks the configuration for synatx errors, but does not commit them.
2. commit confirmed - have you ever locked yourself out of a device because of a change you made? More often than not? Then 'commit confirmed' is for you. This commits the changes, but starts a timer as well (default being 10 minutes) - within this time, the user is expected to 'commit' the changes again, thus confirming the commit. If the user does not commit the changes again, then the configuration is automatically rolled back.
3. commit comment - very git like. You can commit against every commit.
4. commit at - allows for commits to be scheduled.

### Change diffs and rolling back

Let's take SW2, as an example, and make some basic changes.

```
{master:0}[edit]
root# set system host-name SW2

{master:0}[edit]
root# set system root-authentication plain-text-password 
New password:
Retype new password:
```

It is very easy to see what changes are pending to be committed using the 'compare' option. All of this is returned in diff style.

```
root# show | compare 
[edit system]
+  host-name SW2;
+  root-authentication {
+      encrypted-password "$6$im9YvqX2$DdZmTiWqRf2O7HAgjFW9ptFsRAT9KXRELcycjG30BkYFMavce7hqwaiz2HznSHVB4FnJStyVdaMD26P1/8WeL1"; ## SECRET-DATA
+  }

{master:0}[edit]
root# 
```

Remember, these changes are not yet commited - this is simply showing us what was added to the candidate configuration, and what isn't present in the active configuration.

The 'commit confirmed' option sheds light into an AMAZING feature on the OS - rollback. Let's commit our changes first.

```
root# commit 
configuration check succeeds
commit complete

{master:0}[edit]
root@SW2#
```

Junos OS creates has a rolling file structure for 'rollback' files, with upto 49 rollback files being stored (0 through 49 without 0 being the active configuration itself). The first three commits are stored under /config and anything after that is stored under /var/db/config. 

You can also see the full configuration of a rollback file using 'show system rollback <number>'. Put together the 'compare' option from before, and you have a very powerful tool - you can view changes compared to an earlier version of the configuration, and combined with the rollback feature, you can roll back to it.  

So, let's say I didn't like the changes I made, and even though I have commited it, I want to go back to my earlier configuration. I can first check what the difference is between my current/active configuration and the rollback I want to go to. Remember, rollback 0 is my active configuration, so there should be no difference between the two. 

```
root@SW2# show | compare rollback 0 

{master:0}[edit]
```

As expected, we see no difference. Let's try that again with rollback 1 now:

```
root@SW2# show | compare rollback 1  
[edit system]
+  host-name SW2;
+  root-authentication {
+      encrypted-password "$6$im9YvqX2$DdZmTiWqRf2O7HAgjFW9ptFsRAT9KXRELcycjG30BkYFMavce7hqwaiz2HznSHVB4FnJStyVdaMD26P1/8WeL1"; ## SECRET-DATA
+  }
-  commit {
-      factory-settings {
-          reset-virtual-chassis-configuration;
-          reset-chassis-lcd-menu;
-      }
-  }
```

This is comparing the active and the rollback configuration - the '+' indicates what is present in the active configuration, while the '-' indicates what is in the rollback configuration. When you rollback, the changes are not commited - just like before, they are simply a part of the candidate configuration. 

This can be confirmed by doing a compare again, after rolling back:

```
root@SW2# rollback 1 
load complete

{master:0}[edit]
root@SW2# show | compare 
[edit system]
-  host-name SW2;
-  root-authentication {
-      encrypted-password "$6$im9YvqX2$DdZmTiWqRf2O7HAgjFW9ptFsRAT9KXRELcycjG30BkYFMavce7hqwaiz2HznSHVB4FnJStyVdaMD26P1/8WeL1"; ## SECRET-DATA
-  }
+  commit {
+      factory-settings {
+          reset-virtual-chassis-configuration;
+          reset-chassis-lcd-menu;
+      }
+  }
```

This tells us that when we commit these changes, the '-' statements will be removed, and the '+' statements will be added. A very powerful tool indeed!

## Basic Bridging
## Basic Routing

## Troubleshooting toolkit

### Debugging on Junos
### Logging on Junos
### Packet captures on Junos