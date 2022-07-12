---
layout: post
title: "Using Sophos Bootable Anti-Virus on Linux"
author: John Moeller
date: 2022-07-12 11:15:15 -0700
tags: [misartg, sbav, antivirus, "Anti-Virus", linux, xfs]
---

### Background ###

We can use [Sophos Bootable Anti-Virus (SBAV)](https://support.sophos.com/support/s/article/KB-000033800?language=en_US) for off-line scans of our machines. However, in its default mode, SBAV doesn't automatically mount our linux machines' drives even though the software does have the ability to scan them. This shows how we mount the drives manually. 

## Our approach ##

* Create your SBAV media ([USB](https://support.sophos.com/support/s/article/KB-000033912?language=en_US) or [ISO/CD](https://support.sophos.com/support/s/article/KB-000033800?language=en_US) and boot your linux machine to it. 

{% include callout.html content="Definitely start with **a backed up test machine** until you've verified that the process works for you. We borked our first machine trying to figure this out, and it had to be restored from backups. <br /><br /> Be sure to mount your volumes as **read-only**!" type="danger" %}

* From the SBAV main menu, choose **Bash Shell (advanced users only)**

![Image that shows the Bash Shell option on the SBAV main menu](/assets/images/22-07-sbav-linux/misartg-sbav-menu-bash-shell.png)

* Once in the shell, it's time to hunt for the machine's volumes. You're likely going to know how to do this for your systems. Ours generally use LVM, so using the `lvscan` tool was helpful for us in finding the underlying storage devices.

![Image that shows the lvscan output results](/assets/images/22-07-sbav-linux/misartg-sbav-shell-lvscan-results.png)
*In this example, we have volumes at `/dev/vg00/root` and `dev/vg00/swap`.*

* Create mountpoint directories for your volumes, if you have more than one, or even if you simply prefer to. 

![Image that shows the creation of a mountpoint we can use for scanning this machine](/assets/images/22-07-sbav-linux/misartg-sbav-shell-create-mountpoints.png)
*In this example, we created the `/mnt/root` mountpoint.*

* Mount your drives **read-only** to your mountpoints. You **must mount them read-only, or they are likely to be break and not be easily recovered.**. We used the commands `mount -o ro /dev/vg00/root /mnt/root` to mount our machine's storage devices. Run `exit` to return to the SBAV menu. 

![Image that shows us mounting the drive to out mountpoint with the read-only option](/assets/images/22-07-sbav-linux/misartg-sbav-shell-mount-readonly.png)

* Run your scans. For linux machines, we use **Sophos Recommended Scans** -> **Scan for viruses (detect-only)** since it can't really remediate anything on the drives themselves. I'm not sure any other option makes sense. 

![Image that shows us running the detect-only scan](/assets/images/22-07-sbav-linux/misartg-sbav-scan-detect-only.png)

![Image that shows us running the scan running](/assets/images/22-07-sbav-linux/misartg-sbav-scan-running.png)
*Scan running.*

![Image that shows us running the scan after its complete](/assets/images/22-07-sbav-linux/misartg-sbav-scan-complete.png)
*Our scans run pretty quickly, about 3GB/minute on this user's CentOS 7 virtual machine with xfs filesystem.*

* After your scan completes, remove/eject your SBAV media and reboot. 

And in case you were worried that the scan wasn't actually doing anything, we did a test where we placed [the EICAR file](https://www.eicar.org/download-anti-malware-testfile/) in a test user's home directory, and it was successfully detected by SBAV:

![Image that shows SBAV finding the EICAR test file in its scan](/assets/images/22-07-sbav-linux/misartg-sbav-EICAR-found-console.png)
*EICAR test file found.*