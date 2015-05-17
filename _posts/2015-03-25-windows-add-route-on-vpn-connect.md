---
layout: post
title:  "Windows: Add static routes to VPN connection automatically"
date:   2015-03-25 13:53:00
categories: windows
tags: windows vpn netsh
---
{% include JB/setup %}

I have a company VPN connection that I do not want all my traffic to go over the link, only certain netblocks.  In order to do this, you need to:
1. Disable default gateway
2. Create netsh script to add the routes
3. Create a scheduled task to fire the netsh script when the link is connected


1. Disable the default gateway
==============================
![Disable default gateway](/assets/disable_default_gateway.png)


2. Create netsh script to add the routes
========================================

Add in routes as you desire

    interface ipv4
    add route prefix=192.168.23.0/24 interface="My VPN" store=active
    add route prefix=172.16.99.0/24 interface="My VPN" store=active
    exit


3. Create a scheduled task to fire on link up
=============================================

The following command will create the scheduled task (split onto multiple lines for readability)

    schtasks /create /F /TN "VPN Connection Update"
        /TR "netsh -f C:\path\to\VpnRoutes.netsh"
        /SC ONEVENT /EC Application /RL HIGHEST
        /MO "*[System[(Level=4 or Level=0) and (EventID=20225)]] and *[EventData[Data='My VPN']]"

*Warning* The scheduled task will not run when on battery and there is no command line setting for this.  You'll need to go into Task Scheduler and change this under the Conditions tab.

Another, and more flexible route would be to create a powershell script to run on connect and call it with

    Powershell.exe -WindowStyle Hidden -NonInteractive -NoProfile -Command C:\path\to\script.ps1
