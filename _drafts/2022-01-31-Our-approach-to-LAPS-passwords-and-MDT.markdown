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

Luckily, there is a better way. 

## Our approach ##

**Our approach makes use of the [Item-level targeting (ILT) feature of Group Policy Preferences](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn789189(v=ws.11)) to dynamically assess whether or not a machine is still deploying, and if it is, LAPS will not be enabled. Once the machine has completed its deployment, this check will fail, LAPS will become enabled and potentially be in effect.**

Our method also works for any system where the the "Default"/autologon entries in the registry are defined, so it's more general than just MDT, or could be adapted for a different autologon scenario that you might need to account for. The same approach could also be broadened for other purposes where you considered yourself locked in to an ADMX setting, but would like some more dynamic control through ILT's powerful capabilities; you can integrate those targeting features and edit the ADMX's registry entries outside of the ADMX instead. 

### Quickstart ###

If you'd like to replicate our approach and are familiar with the problem and AD/GPOs/ILT/LAPS/MDT, you can follow these steps **first in a test environment or test OU/security-filtered group** before trying it out and re-creating/re-linking more broadly. 

We'll have a step-by-step guide below if you'd like more context, links to resources, some screenshots and some snippets that you may be able to copy/paste into your environment. 

* Load the LAPS ADMX(es) into your AD if you haven't already.
* Build an installer for deploying LAPS to clients. You might do this in MDT, your configuration management system, or in a GPO for `Software Settings` -> `Assigned Applications`. It's an MSI-based installer, so it is relatively simple to deploy.
* Create a new GPO for your LAPS settings (password construction details, minimum & maximum age, etc), but **do not** define the `Enable local admin password management` setting.
* Create another new GPO for dynamic LAPS enablement, and in it:
  * Create a new **Registry Item** to **Update** `HKLM\SOFTWARE\Policies\Microsoft Services\AdmPwd\AdmPwdEnabled` to a `REG_DWORD` value of **`1`**. 
    * In this registry item, visit the **Common** tab, click the checkbox to Enable `Item-level targeting` and enter the `Targeting...` menu. 
      * Create a new **Registry Match** ILT item:
        * `Match type`: Match value data
        * `Value data match type`: Any
        * `HIVE`: HKEY_LOCAL_MACHINE
        * `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
        * `Value name`: **AutoAdminLogon**
        * `Value type`: Any
        * `Value data`: **1**
      * Right-click the ILT item you just created and choose `Item Options` -> **`Is not`**. 
      * Create a second **Registry Match** ILT item, ensure it has an **AND** relationship with your first one, with these settings:
        * `Match type`: Match value data
        * `Value data match type`: Any
        * `HIVE`: HKEY_LOCAL_MACHINE
        * `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
        * `Value name`: **DefaultUserName**
        * `Value type`: Any
        * `Value data`: *the username of the local administrator account that MDT uses*. In our case, this is the default, **Administrator** .
      * Right-click the item you just created and choose `Item Options` -> **`Is not`**.   
  * Create another new **Registry Item** to **Update** `HKLM\SOFTWARE\Policies\Microsoft Services\AdmPwd\AdmPwdEnabled` to a `REG_DWORD` value of **`0`**. 
    * This one should have a higher `Order` number than the previously-created item. 
    * In this registry item, again go to the Common tab, enable Item-level targeting, and in the targeting menu:
      * Create a new **Registry Match** ILT item:
        * `Match type`: Match value data
        * `Value data match type`: Any
        * `HIVE`: HKEY_LOCAL_MACHINE
        * `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
        * `Value name`: **AutoAdminLogon**
        * `Value type`: Any
        * `Value data`: **1**
      * Create a second **Registry Match** ILT item, ensure it has an **AND** relationship with your first one, with these settings:
        * `Match type`: Match value data
        * `Value data match type`: Any
        * `HIVE`: HKEY_LOCAL_MACHINE
        * `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
        * `Value name`: **DefaultUserName**
        * `Value type`: Any
        * `Value data`: *the username of the local administrator account that MDT uses*. In our case, this is the default, **Administrator** .
* Ensure that your GPO for LAPS settings is applied **before** the second GPO for the registry item with the ILT settings.[^fn-precendence] That is, the first GPO made should have a higher precendence number than the second in the `Group Policy Inheritance` tab.
* Test! Check out your in-band/previously built clients to ensure that LAPS is applying to them, then try to test out an MDT build and ensure a domain-joined machine is does not have LAPS enabled during MDT's build process. You can use MDT's **Suspend**/**Resume** pattern to help you accomplish this, along with the **Resultant Set of Policies** snap-in. You can further verify that a particular computer "flips" to enabled LAPS after it finishes its MDT build successfully.

[^fn-precendence]: This isn't *technically* necessary with the design we have, but will be helpful later if someone alters your LAPS GPO and unwittingly enables it across the board. The hide you save may be your own! In the `Precendence` column, the smallest number is applied last. Being mindful and intentional with GPO precendence (and inherited precendence) is a good practice anyway, as the order of GPO/GPP application can overwrite a previous setting, and sometimes this is exactly what you want to accomplish something interesting. That's how the `Enforce` setting works on GPOs as well. 

### Our full step-by-step guide ###

We'll expound on some of our decisions in this guide. 

#### Load the LAPS ADMX(es) into your AD if you haven't already. ####

Like many pieces of software intended to work with Active Directory, or be managed in an enterprise Windows-based environment, LAPS includes its own [Group Policy administrative templates](https://docs.microsoft.com/en-us/troubleshoot/windows-server/group-policy/manage-group-policy-adm-file), often refered to by their file extensions of "ADMX" or "ADM". These files can be imported into your domain controllers and will provide ways to manipulate settings through Group Policy settings, in the `Administrative Templates` area of User and Computer Group Policy Objects. Many third-party software vendors provide adminstrative templates as a convenience. ADMXs ultimately define registry settings, and that's what our method takes advantage of. An excellent resource for discovering which registry settings a given ADMX setting modifies is [admx.help](https://admx.help). You can [browse their page on the LAPS settings](https://admx.help/?Category=LAPS) if you'd like to dive deeper into where they get applied to a system. 

LAPS' .admx files get loaded into your domain the same way that others do. If you're unfamiliar, or you'd just like to see LAPS-specific ADMX loading information, see these links:
* [Wendy Jiang's reply to *GPO ADMX files for LAPS* on TechNet](https://social.technet.microsoft.com/Forums/en-US/0c1e51d9-125f-49d4-9526-8d660eeeb15a/gpo-admx-files-for-laps?forum=winserverGP) 
* [The beginning section on thesysadmins.co.uk's blog on installing LAPS (Part 2)](https://blog.thesysadmins.co.uk/deploying-microsoft-laps-part-2.html)

#### Build an installer for deploying LAPS to clients ####

I am making the assumption that if you're interested in our guide that you probably already a preferred method for deploying software to computers under your management. And if so, you don't need to know much about deploying the LAPS client onto your endpoints other than it includes an .MSI installer files for 64-bit and 32-bit machines and they work as expected. You may have already built an installer on your MDT system that you can use.

You can deploy the LAPS client to your endpoints and it will remain dormant until you define LAPS policies in later steps, and you'll definitely need it to be able to use LAPS, so you'll want to get that squared away now. 

If you don't have a go-to method for deploying LAPS, or would like it perhaps more tied to your Active Directory than other installations (this may be appropriate since LAPS relies on Active Directory), you can build a GPO that installs it for you. You will need to host the .MSI installer files[^fn-32bit] on a share that your computers can reach and read, and you can follow [Prajwal Desai's excellent instructions on how to *Create a GPO to deploy LAPS*](https://www.prajwaldesai.com/how-to-install-and-deploy-microsoft-laps-software/#Create-a-GPO-to-deploy-LAPS).

[^fn-32bit]: You may only need to support the 64-bit installation method if you don't have any 32-bit clients anymore. 

#### Create a new GPO for your LAPS settings ####

Our method will have us making two GPOs where someone would ordinarily create one; one policy to manage the settings for LAPS, and a later policy dynamically disable LAPS while MDT is building, and enable it otherwise.

I *highly suggest* creating your new policy in a test OU or test environment of some kind. You can always re-link or re-create policies later once you've verified that they're working. 

You'll need to decide what LAPS settings make sense for your situation, but here's an example of ours:
![Image that shows partial LAPS settings in an Active Directory environment](/assets/images/misartg-LAPS-settings.png)

If you're using LAPS in earnest, you certainly want to **Enable** the `Do not allow password expiration time longer than required by policy` policy. This ensures that an outside influence cannot extend the LAPS password deadline, which can circumvent the control.

You will also want a short expiration time, defined in `Password Age (Days)`. I think that setting it to the minimum of **1 day** makes most sense. 

Your organization may have guidance or requirements you need to follow with the `Password Complexity` settings.

***Important***: **Do not** define the `Enable local admin password management` setting in this policy; we'll do it in another one.

*Note: I like to mention that this policy is partial in the policy name, as a reminder to myself and other administrators that this policy does not capture all the settings on its own. GPOs are often maintained in collaboration with others and this is a clue to others (or yourself, if it's been a while) that there's something else going on here.*

#### Create another GPO, for dynamic LAPS enablement ####

Now, it's time for the interesting part. 

You want to create a new policy






---
Footnotes:
