---
layout: post
title: "Fixing the Ubuntu/Debian netboot installer to work with our DHCP server"
author: "John Moeller"
date: 2025-09-03 09:09:09 -0700
tags: [ubuntu, debian, network, netboot, dhcp, ip, netboot.xyz]
---

We're usually an Enterprise Linux shop, but we recently wanted to explore the [Proxmox Offline Mirror](https://pom.proxmox.com/) which uses an `apt` package.

We ran into an issue in our environment where the Ubuntu network-based installer, which we were trying to deploy in a VM through the fantastic [netboot.xyz](https://netboot.xyz/) project, would not successfully network boot for us because the installer couldn't get an IP address from our (Windows) DHCP server. Digging into this, we found that the Client Identifier being received by the DHCP server was a very long value that had extraneous characters, whereas the DHCP server seemed to be expecting just a 12-digit MAC address for Client ID, [similar to the issue described in this Spiceworks post](https://community.spiceworks.com/t/strange-extra-long-linux-mac-address-in-dhcp-active-leases/775449). We confirmed we had the same issue with network-based installation of Debian as well.

If you were inside a full Debian-family OS, there's certain config changes or workarounds you can make to easily overcome this, and in fact, we had a note in our troubleshooting documentation that some of our older Ubuntu LTS systems could successfully re-IP just by running `dhclient` on the command-line if they failed to get an IP at boot. But in a pre-OS netboot situation, those aren't options available to you yet. You're limited to supplying kernel paramters to try to overcome the issue. My quick attempts at providing `ip=dhcp` or `ip=bootp` didn't resolve the issue.

[Mercifully, this StackOverflow comment by Oleg Golovanov included the full syntax of the `ip` parameter in a way that the netboot installer could handle](https://stackoverflow.com/a/77733349/23493737). 

By passing **`ip=::::$(hostname)::dhcp:::`** to the Debian and Ubuntu network installers, it sends the Client Identifier in the syntax our DHCP server expects, gets an IP address, and therefore the network installer works.

In our local netboot.xyz's `ubuntu.ipxe` file, we commented out line 141 and replaced it as follows:

```
141 #isset ${dhcp-server} && set netboot_params ip=dhcp url=${ubuntu_iso_url} || set netboot_params
142 isset ${dhcp-server} && set netboot_params ip=::::$(hostname)::dhcp::: url=${ubuntu_iso_url} || set netboot_params
```

I imagine this is a relatively niche issue bourne out of the combination of technologies in the stack we're using, any of which could be at fault or missing correct configuration. But I wanted to write it up since it's one of the "magic commands" that fixed everything for us once we found it, and it's the sort of thing I'm likely to forget about entirely a year hence. 

Thanks Oleg!