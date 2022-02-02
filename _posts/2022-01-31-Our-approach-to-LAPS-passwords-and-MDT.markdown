---
layout: post
title: "Our approach to LAPS passwords and MDT"
author: John Moeller
date: 2022-01-31 10:00:15 -0700
tags: [misartg, LAPS, MDT, GPO, GPP, ILT, "Active Directory"]
---

### Background ###
Our team makes extensive use of Microsoft Deployment Toolkit (MDT) to build and rebuild Windows-based computers and virtual machines. We've been using MDT for many years, and have dozens of complex Task Sequences and templates for handling certain deployment scenarios. 

Recently, we wanted to have a better way for our MDT system to coexist with Microsoft's Local Administrator Password Solution (LAPS). LAPS can be used in Active Directory environments to automate periodic resets of local administrative credentials, like the local Administrator (ie, "SID 500") account. If you'd like to know more about LAPS and some of its intricacies, [Alex Asplund's article is excellent](https://adamtheautomator.com/microsoft-laps/). LAPS settings are managed with Group Policy Objects (GPOs), and LAPS password resets are assessed for need and performed if required by the Group Policy Update process. 

#### MDT with LAPS considered harmful ####

The primary issue with LAPS and MDT alongside one another is that during deployment, if LAPS is active, the password for the Administrator account may be reset by LAPS then MDT will not be able to continue its Task Sequence; the machine expects to be able to log in as Administrator with the credentials that were previously set in the Task Sequence creation, but they've been changed by LAPS. [MDT puts its login information into the "Default" login (sometimes called autologon) areas of the registry](https://docs.microsoft.com/en-us/troubleshoot/windows-server/user-profiles-and-logon/turn-on-automatic-logon) at the start of deployment and will fail to login and continue if they've been changed since then. Where the deployments fail may be inconsistent depending on how long the sequence runs, your LAPS settings, and your luck with timing.

There's several popular strategies for dealing with this:
1. Place the computers you're building in a "staging OU" with blocked inheritance, where the LAPS GPOs do not apply. 
2. In MDT's Task Sequences, move the `Recover From Domain` task, where domain join occurs, to near the end of the sequence.

I don't like these for several reasons:
1. For staging OUs, it can be a bit of bear to re-link policies from above in hierarchy, as this has to be done manually as new policies are created or linked. It can also be a bit of a chore to come back to the staging OU after deployment and move computer objects back to where they should reside in your OU structure; this is especially true in a rebuild scenario where the computer is already where it should be. We also have several long-running deployments and it's easy to forget to move the computer out of staging after its done building. 
2. Moving the `Recover From Domain` task is simple and effective, but if you have many Task Sequences you have to visit each one to change this setting, and you have to remember to move it on newly-created Task Sequences[^fn-mdt-template]. And while it's less than ideal to couple in-sequence actions to settings you get from being domain joined, your deployments may rely on such settings. We take advantage of being domain-joined to register and license certain software based on registry information set from GPPs, and also on the domain's settings for MS licensing and activation.

[^fn-mdt-template]: You can mitigate this somewhat by [creating and using Task Sequence Templates in MDT](https://www.danielengberg.com/how-to-create-an-mdt-task-sequence-template/), where the `Recover From Domain` task is in the correct place. But, of course, you'd have to recreate or alter your existing templates to move them, along with any pre-existing task sequence. But I **highly recommend** using templates in general so you can reuse complex MDT folders and tasks across deployment scenarios. 

## Our approach ##

Luckily, there is a better way. 

**Our approach makes use of the [Item-Level Targeting feature of Group Policy Preferences](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn789189(v=ws.11)) to dynamically assess whether or not a machine is still deploying, and if it is, LAPS will not be enabled.**

**Once the machine has completed its deployment, this check will fail, LAPS will become enabled and potentially be in effect.**

Our approach work for any system where the the "Default"/autologon entries in the registry are defined, so it's more general than just MDT, or could be easily adapted for a different scenario that you might need to account for.






