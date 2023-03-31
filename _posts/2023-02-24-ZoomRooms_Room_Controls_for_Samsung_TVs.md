---
layout: post
title: "Zoom Rooms \"Room Controls\" for Samsung display power actions"
author: John Moeller
date: 2023-02-24 13:05:05 -0700
tags: [zoom, samsung, mdc, "Room Controls", "Zoom Rooms", display, tv, qm-h, power]
---

### Background ###
We wanted to look at controling our Zoom Room TVs, Samsung commercial displays, using the Zoom Rooms "Native Room Controls" method, [since Zoom says that the older HDMI CEC method for turning on and off the display is no longer supported](https://support.zoom.us/hc/en-us/articles/115003340906-Zoom-Rooms-display-systems-on-off). 

The original Zoom "Device Operation Time" feature mostly worked for us[^fn-duke-hdmicec] on our simple Zoom Rooms, but the new Room Controls features would allow us to do more interesting things with other components down the line.

[^fn-duke-hdmicec]: I suspect HDMI CEC would have worked fully for us if we had read and followed the [Duke DDMC's fantastic guide](https://sites.duke.edu/ddmc/2019/01/17/zoom-room-tv-control-a-cec-story/) when we got started, instead of finding it after the fact.  

## Our approach ##
Please see [our gist](https://gist.github.com/jmoeller-ua/a8c1eed5634bf00f0a1fafce7dcd303d) for a configuration sample JSON that works with our Samsung QM-H series displays, using [Samsung's MDC service](https://displaysolutions.samsung.com/fileDownload/21891). 

{% gist a8c1eed5634bf00f0a1fafce7dcd303d %}

### Helpful links ###
* The excellent [Samsung-MDC](https://github.com/vgavro/samsung-mdc) project helped us in all phases, from initial understanding of the MDC service's capabilities, testing, and then later on configuration details in the Room Controls JSON. Comparing the `samsung-mdc` verbose output and source code for individual commands with Wireshark captures allowed us to see that we weren't properly escaping our hex codes. This is an excellent tool that we will use for additional non-Zoom display control going forward. 

* [Just Add Power](http://justaddpower.com/)'s comprehensive Samsung RS232 support pages for [RS232C](https://support.justaddpower.com/kb/article/245-samsung-rs232-control-rs232c/) and [Ex-Link](https://support.justaddpower.com/kb/article/16-samsung-rs232-control-exlink/) formats confirmed we were on the right track and will be most  helpful if we ever need to manipulate other Samsungs via Ex-Link. 

* [Control Concepts](https://controlconcepts.net) has a [Room Controls profile maker](https://controlconcepts.net/product/zoom-profile-maker/) and [an excellent guide](https://controlconcepts.net/zoom/assets/pdf/cci-zoom-room-controls-profile-maker-help.pdf) that helped us understand how to format our command output as hex (though it appears you need to escape it with another `\` when pasting it into the Zoom Room's settings), and how to chain commands with parameters with the `%` symbol. 


---
### Footnotes ###