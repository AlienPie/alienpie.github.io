---
title: IOS-XR Upgrades – PIEs, SMUs and Certificate Expirations
description: I recently had to upgrade an ASR9001 but found myself resolving a Cisco Field notice which prevented the installation of updates...
author: "Stan Jewhurst"
date: 2019-05-10T12:05:51+00:00
#url:
cover:
  image: images/cover-image.png
  #caption: A featured image!
tags:
  - asr9k
  - ios-xr
  - networking
ShowToc: false
draft: false
---
Given that these devices tend to be the most stable and unchanging in our network (and in most networks) it's rare I have to upgrade and ASR9000 Router. Every time I approach it I realise I've completely forgotten the process, so hopefully this will help me remember in the future.

IOS-XR is modular software, based on a high-performance Linux kernel. This means that, after the base software features have been installed, you can customise with just the features you want, to keep your install minimalistic and to also stop you encountering bugs with software features you're not even using.

IOS-XR uses two types of software package - a PIE (Package Installation Envelope) and a SMU (Software Maintenance Upgrade). The PIEs are used to install main software features, such as the core IOS-XR Operating System and then feature sets such as multicast, MPLS and cryptographic functions. SMUs are a pretty novel approach to bug-fixing; In the past you would need to upgrade your entire OS to a maintenance release in order to get the latest bug-fixes. With IOS-XR, you just install a SMU, which patches the bug (usually) and doesn't require you to reload your device (again, usually).

There is a third type of package available called an SP (Service Pack) which Cisco release on a relatively regular basis, which bundles up all the SMUs released up to a given point in time, and allows you to install them all together. Today, I'm upgrading an ASR9001 from 5.1.0 to 6.1.4.

The first step is getting the software onto the device - easy enough if you have a FTP Server or a (\*shivers\*) TFTP server in your network. Collect all your PIEs together into a folder and get them onto the device:

```
RP/0/RSP0/CPU0:ASR-9001#copy ftp://username:password:@10.0.10.1;mgmt/asr9k-mini-px.pie-6.1.4
CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
477999627 bytes copied in    375 sec (  1273283)bytes/sec
```

I've only shown the first PIE above but you'll need to bring all your packages across. Once all your images are on the device (you can confirm this with a **`dir disk0:`** command) it's time to start the process of installing them. Installation of IOS-XOR PIEs comes in three steps:

  * **`add`** the PIE
  * **`activate`** the PIE
  * **`commit`** the PIE

Each step performs a number of checks, including verifying the image, unpacking it and then making it active at next boot (we're going to skip ISSU as it's not supported in 5.1.0 and finally committing all the changes (which usually reloads the device).

It was here I ran into my first issue. I ran the add command for my mini-px PIE (The mini PIE contains all the base OS functions - its the smallest package that renders a usable IOS-XR install) and all seemed to be going well:

```
RP/0/RSP0/CPU0:ASR-9001(admin)#install add disk0:/asr9k-mini-px.pie-6.1.4
Fri May 10 08:12:01.690 UTC
Install operation 18 '(admin) install add /disk0:/asr9k-mini-px.pie-6.1.4' started by user 'stan' via CLI at 08:12:02 UTC Fri May 10 2019.
The install operation will continue asynchronously.
```

The process continued to run in the background (you can check the progress with the **`show install requests`** command) until:

```
 Error:    Cannot proceed with the add operation because the code signing certificate has
 Error:    expired.
 Error:    Suggested steps to resolve this:
 Error:     - check the system clock using 'show clock' (correct with 'clock set' if
 Error:       necessary).
 Error:     - check the pie file was built within the last 5 years using '(admin) show
 Error:       install pie-info /disk0:/asr9k-mini-px.pie-6.1.4'.
```

Well at least it's a somewhat useful error. I checked the clock - all good, synced to NTP. I checked the PIE, also fine. So what could it be? Time to consult google, and instantly got a hit - [Field Notice 63979][1]. I was actually aware of this issue as it had come to light when I was working with a large ISP in the past. Essentially, in October 2015, the Software Signing certificate Cisco bundled in with IOS-XR expired - meaning if you tried to install software after that time it would be unable to verify the software as genuine (much like when you try to visit a website when its HTTPS certificate has expired). PIE, SMU and SP installations would all fail. At the time your could install a SMU before the expiration date in order to renew the root certificate, but it's now 2019.

The Field Notice gave clear instruction that the issue could be resolved by installing a new root certificate and then installing a post-expiry SMU. However, you have to look for the cryptic link in the middle of the article to get the [instructions][2] on how to do this. So cue downloading the SMU, un-taring it and FTP-ing the files onto the disk. Once these were in place, to install the root certificate, you have to drop to a shell on IOS-XR (remember, it's based on Linux)

```
RP/0/RSP0/CPU0:ASR-9001#run
Fri May 10 08:58:26.360 UTC
#samcmd sam add certificate /disk0:/css-root.cer root trust
SAM: Successful adding certificate /disk0:/css-root.cer
exit
RP/0/RSP0/CPU0:ASR-9001#admin
Fri May 10 08:58:48.636 UTC
RP/0/RSP0/CPU0:ASR-9001(admin)#install add disk0:/asr9k-px-5.1.0.CSCut52232.pie sync
Fri May 10 08:59:08.223 UTC
Install operation 21 '(admin) install add /disk0:/asr9k-px-5.1.0.CSCut52232.pie synchronous' started  by user 'stan' via CLI at 08:59:08 UTC Fri May 10 2019.
 Info:     The following package is now available to be activated:
 Info:
 Info:         disk0:asr9k-px-5.1.0.CSCut52232-1.0.0
 Info:
 Info:     The package can be activated across the entire router.
 Info:
 / 100% complete: The operation can no longer be aborted (ctrl-c for options)
 Install operation 21 completed successfully at 08:59:19 UTC Fri May 10 2019.
```

Excellent, now on with adding packages!

```
RP/0/RSP0/CPU0:ASR-9001(admin)#install add disk0:/asr9k-mini-px.pie-6.1.4
Fri May 10 09:02:48.699 UTC
Install operation 23 '(admin) install add /disk0:/asr9k-mini-px.pie-6.1.4' started by user 'stan' via CLI at 09:02:48 UTC Fri May 10 2019.
The install operation will continue asynchronously.
RP/0/RSP0/CPU0:ASR-9001(admin)#Info: The following package is now available to be activated:
 Info:
 Info: disk0:asr9k-mini-px-6.1.4
 Info:
 Info: The package can be activated across the entire router.
 Info:
 Install operation 23 completed successfully at 09:19:49 UTC Fri May 10 2019.
```

Once all the packages are added, the next step is activating them. There are a few caveats to this step - firstly you need to make sure you've installed new versions of all your currently running packages - you can confirm this with the **`show install active`** and **`show install inactive`** commands and compare the lists.

```
RP/0/RSP0/CPU0:ASR-9001(admin)#show install active
 Fri May 10 10:02:34.447 UTC
 Secure Domain Router: Owner

Node 0/RSP0/CPU0 [RP] [SDR: Owner]
     Boot Device: disk0:
     Boot Image: /disk0/asr9k-os-mbi-5.1.0/0x100000/mbiasr9k-rp.vm
     Active Packages:
       disk0:asr9k-fpd-px-5.1.0     <==
       disk0:asr9k-k9sec-px-5.1.0   <==
       disk0:asr9k-mgbl-px-5.1.0    <==
       disk0:asr9k-mini-px-5.1.0    <==
       disk0:asr9k-mpls-px-5.1.0    <==
       disk0:asr9k-px-5.1.0.CSCut52232-1.0.0
```

```
RP/0/RSP0/CPU0:ASR-9001show install inactive
 Fri May 10 10:43:11.576 UTC
 Secure Domain Router: Owner

Node 0/RSP0/CPU0 [RP] [SDR: Owner]
     Boot Device: disk0:
     Inactive Packages:
       disk0:asr9k-mini-px-6.1.4    <==
       disk0:asr9k-fpd-px-6.1.4     <==
       disk0:asr9k-mgbl-px-6.1.4    <==
       disk0:asr9k-mpls-px-6.1.4    <==
       disk0:asr9k-mcast-px-6.1.4
       disk0:asr9k-k9sec-px-6.1.4   <==
```

I'm adding the MPLS PIE to this upgrade as well. Once we've confirmed, we can activate the packages. They need to be activated together, to ensure all the components are updated as well:

```
RP/0/RSP0/CPU0:ASR-9001(admin)#install activate disk0:asr9k-mini-px-6.1.4 disk0:asr9k-mpls-px-6.1.4 disk0:asr9k-fpd-px-6.1.4 disk0:asr9k-k9sec-px-6.1.4 disk0:asr9k-mcast-px-6.1.4 disk0:asr9k-mgbl-px-6.1.4
Install operation 36 '(admin) install activate disk0:asr9k-mini-px-6.1.4 disk0:asr9k-mpls-px-6.1.4 disk0:asr9k-fpd-px-6.1.4 disk0:asr9k-k9sec-px-6.1.4 disk0:asr9k-mcast-px-6.1.4
 disk0:asr9k-mgbl-px-6.1.4' started by user 'stan' via CLI at 10:44:59 UTC Fri May 10 2019.
 Info:     This operation will reload the following nodes in parallel:
 Info:         0/RSP0/CPU0 (RP) (SDR: Owner)
 Info:         0/0/CPU0 (LC) (SDR: Owner)
 Proceed with this install operation (y/n)? [y]
 Info:     Install Method: Parallel Reload
 The install operation will continue asynchronously.
RP/0/RSP0/CPU0:May 10 10:49:45.851 : instdir[259]: %INSTALL-INSTMGR-7-SOFTWARE_CHANGE_RELOAD : Software change transaction 36 will reload affected nodes
```

The install will continue and give you a few heads up and messages as it carries out the various tasks and then finally reload. When it comes pack we can see that the new software packages are active:

```
RP/0/RSP0/CPU0:ASR-9001#sh install active
Fri May 10 11:05:52.542 UTC
Secure Domain Router: Owner

  Node 0/RSP0/CPU0 [RP] [SDR: Owner]
    Boot Device: disk0:
    Boot Image: /disk0/asr9k-os-mbi-6.1.4/0x100000/mbiasr9k-rp.vm
    Active Packages:
      disk0:asr9k-mini-px-6.1.4
      disk0:asr9k-fpd-px-6.1.4
      disk0:asr9k-mgbl-px-6.1.4
      disk0:asr9k-mpls-px-6.1.4
      disk0:asr9k-mcast-px-6.1.4
      disk0:asr9k-k9sec-px-6.1.4
```

Finally, we commit the software packages:

```
RP/0/RSP0/CPU0:ASR-9001(admin)#install commit
Fri May 10 11:26:16.623 UTC 
Install operation 1 '(admin) install commit' started by user 'stan' via CLI at 11:26:17 UTC Fri May 10 2019.

100% complete: The operation can no longer be aborted (ctrl-c for options)

RP/0/RSP0/CPU0:May 10 11:26:30.523 : instdir[256]: %INSTALL-INSTMGR-4-ACTIVE_SOFTWARE_COMMITTED_INFO : The currently active software is now the same as the committed software.

Install operation 1 completed successfully at 11:26:30 UTC Fri May 10 2019. 
```

And done! We've not got an ASR9001 running a fresh install of 6.1.4.

> EDIT: Just a final note on this. The package actually contains upgrade code for the FPDs on the ASR9001 i.e. Microcode upgrade for the FPGAs themselves.

```
RP/0/RSP0/CPU0:ASR-9001#admin show hw-module fpd location 0/RSP0/CPU0

===================================== ==========================================
                                       Existing Field Programmable Devices
                                       ==========================================
                                         HW                       Current SW Upg/
 Location     Card Type                Version Type Subtype Inst   Version   Dng?
 ============ ======================== ======= ==== ======= ==== =========== ====
 0/RSP0/CPU0  ASR9001-RP                 1.0   lc   fpga2   0       1.14     Yes
                                               lc   cbc     0      22.114    No
                                               lc   rommon  0       2.01     Yes
---------------------------------------------------------------------------------
 NOTES:
 One or more FPD needs an upgrade.  This can be accomplished
 using the "admin> upgrade hw-module fpd  location " CLI. 
```

This can be achieved with the command **`admin upgrade hw-module fpd all location 0/RSP0/CPU0`** followed by a reload.

 [1]: https://www.cisco.com/c/en/us/support/docs/field-notices/639/fn63979.html
 [2]: https://www.cisco.com/c/en/us/td/docs/routers/technotes/MOP-CSS-to-Abraxas.html
