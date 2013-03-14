---
layout: post
title: "/etc/network/interfaces Ubuntu Linux networking example"
date: 2013-03-14 14:19
comments: true
categories: ubuntu
---

**Q.** Can you explain how to setup network parameters such as IP address, subnet, dhcp etc using /etc/network/interfaces file?

**A.** /etc/network/interfaces file contains network interface configuration information for the both Ubuntu and Debian Linux. This is where you configure how your system is connected to the network.
<!--more-->

## Defining physical interfaces such as eth0
Lines beginning with the word "auto" are used to identify the physical interfaces to be brought up when ifup is run with the -a option. (This option is used by the system boot scripts.) Physical interface names should follow the word "auto" on the same line. There can be multiple "auto" stanzas. ifup brings the named inter faces up in the order listed. For example following example setup eth0 (first network interface card) with 192.168.1.5 IP address and gateway (router) to 192.168.1.254:  
<code>
iface eth0 inet static  
address 192.168.1.5  
netmask 255.255.255.0  
gateway 192.168.1.254  
</code>

## Setup interface to dhcp
To setup eth0 to dhcp, enter:  
<code>
auto eth0  
iface eth0 inet dhcp  
</code>

## Examples: How to set up interfaces
Please read our previous  
[How to: Ubuntu Linux convert DHCP network configuration to static IP configuration for more information.](http://www.cyberciti.biz/tips/howto-ubuntu-linux-convert-dhcp-network-configuration-to-static-ip-configuration.html)

Following is file located at **/usr/share/doc/ifupdown/examples/network-interfaces**, use this file as reference (don't forget interfaces man pages for more help):  
{% include_code ubuntu/network-interfaces.cfg %}

via [http://www.cyberciti.biz/faq/setting-up-an-network-interfaces-file/](http://www.cyberciti.biz/faq/setting-up-an-network-interfaces-file/)

