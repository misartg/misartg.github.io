---
layout: post
title: "Expire-LAPSPassword.ps1"
author: John Moeller
date: 2023-09-19 14:35:35 -0700
tags: [LAPS, expire, expiration, password, ms-MCS-AdmPwdExpirationTime]
---

Someone contacted me with some questions about Windows LAPS from our [MDT+LAPS post](/2022/02/08/Our-approach-to-LAPS-and-MDT.html), and we thought they might benefit from seeing the internal PowerShell script we wrote to expire the local machine's LAPS password, Expire-LAPSPassword.ps1. So I've put it in [a gist](https://gist.github.com/jmoeller-ua/9314a6e2bb7c55a93855b865938385cf). 

{% gist 9314a6e2bb7c55a93855b865938385cf %}

### Notes ###

- This script has a dependency on [Invoke-CommandAs](https://github.com/mkellerman/Invoke-CommandAs), since we need to run the expiration command as the `SYSTEM` user. It's an excellent PS module for this purpose and I'm grateful for its existence. 

- I cannot recall exactly why we had to use the 2.8.5.201 version of the NuGet provider, but I remember it seemed important at the time. That module-loading stanza has been copy-pasted into over a dozen of our PS scripts over the years, so it's likely whatever reason that was may not be applicable any more. 

- This script is a bit chatty in terms of output, which met our needs. If you don't like that, you could easily change some of the `Write-Output` commands to `Write-Verbose` to quiet it down some. 

- I'll glady take feedback if you have it. When it comes to scripting and coding, I'm an overconfident amateur. I'd love to continue improving with your tips. 

---

