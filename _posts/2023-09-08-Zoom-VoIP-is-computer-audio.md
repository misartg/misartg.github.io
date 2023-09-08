---
layout: post
title: "In Zoom's world, VoIP = Computer Audio"
author: John Moeller
date: 2023-09-08 10:05:05 -0700
tags: [zoom, gpo, admx, voip, "Computer Audio"]
---

I was looking at setting some default settings for Zoom in our computer lab using the very helpful "Recommendation Setting" options from [their ADMX](https://support.zoom.us/hc/en-us/articles/360039100051-Mass-deploying-with-Group-Policy-Objects), but was struggling to find the setting to default to "Join with Computer Audio". 

It turns out in Zoom's lingo, this is called "Auto connect audio with VoIP when joining meeting"
![Image that shows admx.help's description of the "Auto connect audio with VoIP when joining meeting (Recommendation Setting)" entry](/assets/images/23-09-zoom-admx/misartg-zoom-admx-auto-connect-audio-with-voip.png)

Setting this to "Enabled" indeed defaulted the scoped computer to Join with Computer Audio. 

The "Recommendation Settings" appear to have worked like how I hoped they would: they applied a default, but if the user changed the setting, the user's setting won and didn't get reset. 


---


{% include callout.html content="[admx.help](https://admx.help) is an excellent resource for those who either need to implement ADMXs through registry GPPs, or just want to browse to see what's in an ADMX before loading it into your Active Directory. They have many relevant 3rd-party ADMXs, including Zoom's." type="primary" %}


