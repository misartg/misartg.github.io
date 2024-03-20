---
layout: post
title: "Our approach to LAPS + MDT"
author: John Moeller
date: 2022-02-08 13:51:15 -0700
last_modified_at: 2023-04-19 15:05:05 -0700
tags: [misartg, LAPS, MDT, GPO, GPP, ILT, "Active Directory"]
---

{% include callout.html content="UPDATE - March 20, 2024: We have tweaked [our updated guidance](#update-41923---updating-our-approach-for-windows-laps) thanks to a wise suggestion from a reader, Jan Inge. Thanks Jan!" type="primary" %}

{% include callout.html content="UPDATE - April 19, 2023: [See our updated guidance](#update-41923---updating-our-approach-for-windows-laps) given the [new Windows LAPS feature included with the April 2023 updates](https://techcommunity.microsoft.com/t5/windows-it-pro-blog/by-popular-demand-windows-laps-available-now/ba-p/3788747)." type="primary" %}

{% include callout.html content="Understand the issue and want to cut straight to \"the recipe\"? See our [Quickstart guide](#quickstart) and save more time by [copying and pasting our GPP Registry items XML](#xml-export-of-my-dynamic-laps-enablement-registry-items)." type="success" %} 

### Background ###

Our team makes extensive use of Microsoft Deployment Toolkit (MDT) to build and rebuild Windows-based computers and virtual machines. We've been using MDT for many years, and have dozens of complex Task Sequences and templates for handling complicated deployment scenarios. 

Recently, we wanted to have a better way for our MDT system to coexist with Microsoft's Local Administrator Password Solution (LAPS). LAPS can be used in Active Directory environments to automate periodic resets of local administrative credentials, like the local Administrator (ie, "SID 500") account. If you'd like to know more about LAPS and some of its intricacies, [Alex Asplund's article is excellent](https://adamtheautomator.com/microsoft-laps/). LAPS settings are managed with Group Policy Objects (GPOs), and LAPS password resets are assessed for need and performed if required by the Group Policy Update process. 

#### MDT with LAPS considered harmful ####

The primary issue with LAPS and MDT alongside one another is that during deployment, if LAPS is active, the password for the Administrator account may be reset by LAPS, then MDT will not be able to continue its Task Sequence; the machine expects to be able to log in as Administrator with the credentials that were previously set in the Task Sequence creation, but they've since been changed by LAPS. [MDT puts its login information into the "Default" login areas (sometimes called autologon) of the registry](https://docs.microsoft.com/en-us/troubleshoot/windows-server/user-profiles-and-logon/turn-on-automatic-logon) at the start of deployment and will fail to login and continue if they've been changed after. Where the deployments fail may be inconsistent depending on how long the sequence runs, your LAPS settings, and your luck with Group Policy Update timing.

There's several popular strategies for dealing with this:
1. Place the computers you're building in a "staging OU" with blocked inheritance, where the LAPS GPOs do not apply. 
2. In MDT's Task Sequences, move the `Recover From Domain` task, where domain join occurs, to near the end of the sequence.

I don't like these for several reasons:
1. For staging OUs, it can be a bit of bear to re-link policies from above in hierarchy and keep that maintained over time, as new policies are created or linked. It can also be a bit of a chore to come back to the staging OU after deployment and move computer objects back to where they should permanently reside in your OU structure; this is especially true in a rebuild scenario where the computer is already where it should be. We also have several long-running deployments and it's easy to forget to move the computer out of staging after its done building. 
2. Moving the `Recover From Domain` task is simple and effective, but if you have many Task Sequences you have to visit each one to change this setting, and you have to remember to move it on newly-created Task Sequences[^fn-mdt-template]. And while it's less than ideal to couple in-sequence actions to settings you get from being domain joined, your deployments may rely on such settings. We take advantage of being domain-joined to register and license certain software based on registry information set from GPPs, and also on the domain's settings for MS licensing and activation.

[^fn-mdt-template]: You can mitigate this somewhat by [creating and using Task Sequence Templates in MDT](https://www.danielengberg.com/how-to-create-an-mdt-task-sequence-template/), where the `Recover From Domain` task is in the correct place. But, of course, you'd have to recreate or alter your existing templates to move them, along with any pre-existing task sequence. But I **highly recommend** using templates in general so you can reuse complex MDT tasks and task folders across deployment scenarios.  

Luckily, there is a better way. 

## Our approach ##

{% include callout.html content="**Our approach makes use of the [Item-level targeting (ILT) feature of Group Policy Preferences (GPP)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn789189(v=ws.11)) to dynamically assess whether or not a machine is still deploying, and if it is, LAPS will not be enabled. Once the machine has completed its deployment, this check will fail, our policy will enable LAPS and LAPS will go into effect.**" type="primary" %} 

Our method also works for any system where the "Default"/autologon entries in the registry are active for the local admin user, so it's more general than just MDT, or could be easily adapted for a different autologon scenario that you might need to account for, or you could make it dynamic based on just about anything that ILT can target. 

This approach could be broadened to other situations where you want more control of application over an ADMX's setting, through ILT's powerful capabilities; you could use GPP's Registry items and item-level targeting features to edit the ADMX's registry entries directly, instead of through the ADMX/GPO interface.

### Quickstart ###

If you'd like to replicate our approach and are familiar with the issue and AD/GPOs/ILT/LAPS/MDT, you can follow these steps **in a test environment or test OU/security-filtered group**. If successful, you could implement it broadly by re-creating/re-linking the policies as needed. 

We'll have a detailed step-by-step guide below if you'd like more context, links to resources and some screenshots. If you're new to LAPS or Group Policy Preference's Item-level targeting feature, you may wish to follow the longer guide. You also reference the longer guide if you want more information about a paricular step mentioned here. 

* Load the LAPS ADMX(es) into your AD if you haven't already[^fn-lapsextendschema].
* Build an installer for deploying LAPS to clients. You might do this in MDT, your configuration management system, or in a GPO for `Software Settings` -> `Assigned Applications`. It's an MSI-based installer, so it is relatively simple to deploy silently.
* Create a new GPO for your LAPS settings (password construction details, password age, etc), but **do not** define the `Enable local admin password management` setting.
* Create another new GPO for dynamic LAPS enablement and either [copy/paste my Registry items XML output below](#xml-export-of-my-dynamic-laps-enablement-registry-items), or make these settings yourself:
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
  * 04/19/23 update for Windows LAPS: Create a new **Registry Item** to **Delete** registry value `BackupDirectory` in `HKLM\Software\Microsoft\Windows\CurrentVersion\LAPS\Config`. 
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
  * 04/19/23 update for Windows LAPS: Create another new **Registry Item** to **Update** registry value `BackupDirectory` in`HKLM\Software\Microsoft\Windows\CurrentVersion\LAPS\Config` to a `REG_DWORD` value of **`0`**. 
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
* Recommended: Order your GPOs so the LAPS settings GPO is applied **before** the second GPO dynamic LAPS enablement.[^fn-precedence] That is, the first GPO made should have a higher precedence number than the second in the `Group Policy Inheritance` tab.
* Test! Check out your in-band/previously built clients to ensure that LAPS is applying to them, with `regedit` and/or the `LAPS UI` application. Then try to test out an MDT build and ensure a domain-joined machine is does not have LAPS enabled during MDT's build process. You can retry problematic deployments and see if they work, or [use MDT's suspend/resume pattern](https://www.toddlamothe.com/deployment/pause-task-sequence-mdt-2010.htm) in a Task Sequence to help you accomplish this, verifying again with `regedit` or the `LAPS UI` application. You can further verify that a particular computer "flips" to enabled LAPS after it finishes its MDT build successfully.

[^fn-precedence]: This isn't *technically* necessary with the design we have, but will be helpful later if someone alters your LAPS GPO and unwittingly enables it across the board. The hide you save may be your own! **Precedence** is the order that the GPOs apply in. In the `Linked Group Policy Objects` and `Group Policy Inheritance` columns, **the smallest number is applied last**[^fn-precedence-vs-gpp-order]. Being mindful and intentional with GPO precedence (and inherited precedence) is a good practice anyway, as the order of GPO/GPP application can overwrite a previous setting, and sometimes this is exactly what you want to accomplish something interesting. Manipulating precedence is how the `Enforce` setting works on GPOs as well.

[^fn-precedence-vs-gpp-order]: Yes, it's annoying and confusing that meaning of the numbers in GPO *precedence* and GPP *order* are reversed from one another!

### Full step-by-step guide ###

We'll expound on some of our decisions in this guide. And this guide might be userful for you if you're new to LAPS, Group Policy Preferences or the Item-level targeting features. You may also be interested in some of our [footnotes](#footnotes).

#### Load the LAPS ADMX(es) into your AD if you haven't already. ####

Like many pieces of software intended to work with Active Directory, or be managed in an enterprise Windows-based environment, LAPS includes its own [Group Policy administrative templates](https://docs.microsoft.com/en-us/troubleshoot/windows-server/group-policy/manage-group-policy-adm-file), often refered to by their file extensions of "ADMX" or "ADM". These files can be imported into your domain controllers and will provide ways to manipulate settings through Group Policy settings, in the `Administrative Templates` area of User and Computer Group Policy Objects. Many third-party software vendors provide adminstrative templates as a convenience. ADMXs ultimately define registry settings, and that's what our method takes advantage of. An excellent resource for discovering which registry settings a given ADMX setting modifies is [admx.help](https://admx.help). You can [browse their page on the LAPS settings](https://admx.help/?Category=LAPS) if you'd like to dive deeper into where they get applied to a system. 

LAPS' .admx files get loaded into your domain the same way that others do. If you're unfamiliar, or you'd just like to see LAPS-specific ADMX loading information, see these links:
* [Wendy Jiang's reply to *GPO ADMX files for LAPS* on TechNet](https://social.technet.microsoft.com/Forums/en-US/0c1e51d9-125f-49d4-9526-8d660eeeb15a/gpo-admx-files-for-laps?forum=winserverGP) 
* [The beginning section on thesysadmins.co.uk's blog on installing LAPS (Part 2)](https://blog.thesysadmins.co.uk/deploying-microsoft-laps-part-2.html)

There are other steps necessary to fully prepare your Active Directory environment for LAPS if it hasn't been already. For more information, see the footnote[^fn-lapsextendschema]. 

[^fn-lapsextendschema]: There may be other steps necessary to prepare your Active Directory for LAPS if it hasn't been already, including extending the schema for new LAPS-related computer object attributes and configuring permissions for computers and users to interact with the new LAPS-related attributes, defining who can read the LAPS password itself. This is a bit outside of the scope of the article, and covered in many other guides, [and is explained quite succinctly in this Now Micro blog post](https://blog.nowmicro.com/2018/02/28/configuring-laps-part-1-configuring-active-directory/). You will have to make your own decisions on which users should be able to read LAPS passwords, and at what level such privileges that should be delegated. 

#### Build an installer for deploying LAPS to clients ####

I am making the assumption that if you're interested in our guide that you probably already a preferred method for deploying software to computers under your management. And if so, you don't need to know much about deploying the LAPS client onto your endpoints other than it includes an .MSI installer files for 64-bit and 32-bit machines and they can be configured to install silently as expected. You could easily create a silent installer to be deployed by MDT if you like. 

You can deploy the LAPS client to your endpoints and it will remain dormant until you define LAPS policies in later steps, and you'll definitely need it to be able to use LAPS, so you'll want to get that squared away now. 

If you don't have a go-to method for deploying LAPS, or would like it perhaps more tied to your Active Directory than other installations (this may be appropriate since LAPS relies on Active Directory), you can build a GPO that installs it for you. You will need to host the .MSI installer files[^fn-32bit] on a share that your computers can reach and read, and you can follow [Prajwal Desai's excellent instructions on how to *Create a GPO to deploy LAPS*](https://www.prajwaldesai.com/how-to-install-and-deploy-microsoft-laps-software/#Create-a-GPO-to-deploy-LAPS).

[^fn-32bit]: You may only need to support the 64-bit installation method if you don't have any 32-bit clients anymore. 

#### Create a new Group Policy Object (GPO) for your LAPS settings ####

Our method will have us making two GPOs where someone would ordinarily create one; one policy to manage the settings for LAPS, and a later policy dynamically disable LAPS while MDT is building, and enable it otherwise.

{% include callout.html content="I *highly recommend* creating your new policy in a test OU or test environment of some kind. You can always re-link or re-create policies later once you've verified that they're working." type="warning" %}

You'll need to decide what LAPS settings make sense for your situation, but here's an example of ours:
![Image that shows partial LAPS settings in an Active Directory environment](/assets/images/22-01-mdt-laps/misartg-LAPS-settings.png)

* If you're using LAPS in earnest, you certainly want to **Enable** the `Do not allow password expiration time longer than required by policy` policy. This ensures that an outside influence cannot extend the LAPS password deadline to circumvent the control.

* You will also want a short expiration time, defined in `Password Age (Days)`. I think that setting it to the minimum, **1 day**, makes most sense[^fn-passwordagedisclaimer]. 

[^fn-passwordagedisclaimer]: Because LAPS password changes are managed by the Group Policy refresh/update (ie, `gpupdate`) process, you can expect a bit of drift from your intended password age. The [background refresh of Group Policy occurs every 90 minutes by default, and may tack on another 0-30 minutes at random on top of that](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/policy/background-refresh-of-group-policy). On each run, group policy update will patrol for an expired LAPS password, changing it if its expired. So the effective age is more like your `Password Age (Days)` value plus some number of minutes up to 120. 

* Your organization may have guidance or requirements you need to follow in the `Password Complexity` settings.

* If your environment uses a different account for local administrator, you can define it in `Name of administrator account to manage`. But if you use the default account **Administrator**, then don't define that setting and LAPS will handle it for you. 

{% include callout.html content="**Do not** define the `Enable local admin password management` setting in this policy; we'll do it in another one." type="danger" %}

{% include callout.html content="I like to mention that this policy is partial in the GPO's name, as a reminder to myself and other administrators that this policy does not capture all the settings on its own. GPOs are often maintained in collaboration with others and this is a clue to others (or yourself, if it's been a while) that there's something else going on here." type="primary" %}

#### Create another GPO, for dynamic LAPS enablement ####

Finally, it's time for the interesting part. Remember that our overall goal is to disallow LAPS to take effect while a computer is being built by MDT, and to enable it otherwise. We're going to manage this with information on the state of the computer from the registry, taking advantage of the Item-level Targeting feature of Group Policy Preferences, and apply LAPS enable/disable setting directly in the registry instead of through the ADMX.[^fn-gpphistory]

[^fn-gpphistory]: Some of us more seasoned types can still recall that before it was Group Policy Preferences, it was PolicyMaker from Desktop Standard. It struck me while writing this article that it's been over 15 years since Microsoft acquired Desktop Standard and PolicyMaker's functionality is still relegated to a separate top-level folder in the GPMC editor window, and not integrated more broadly. But I should probably just be happy that it's there at all as it continues to be a powerful tool that we rely on for all sorts of advanced configuration that would otherwise be hidden away in scripts and configuration management platforms. 

To do this, we need to gather some information.

First, you need to know the username of the account that MDT uses for local admin during deployments. The overwhelming majority of the time, this is the default, built-in **Administrator** account, but if you've changed it somehow, you'd need to know its username. But I'll be writing the rest of the guidance assuming it is **Administrator**. 

Second, we need to know the actual registry setting that the LAPS Administrative Template setting modifies to enable or disable LAPS. Luckily, [the fantastic admx.help has a page that can tell us what it is](https://admx.help/?Category=LAPS&Policy=FullArmor.Policies.C9E1D975_EA58_48C3_958E_3BC214D89A2E::POL_AdmPwd_Enabled).

![Image that shows the registry setting for enabling/disabling LAPS, which is HKLM\	Software\Policies\Microsoft Services\AdmPwd\AdmPwdEnabled, using a value of 1 for enabled](/assets/images/22-01-mdt-laps/misartg-LAPS-admxhelp-enable-LAPS-reg.png)

The registry setting to enable or disable LAPS is a DWORD value of `AdmPwdEnabled` at `HKEY_LOCAL_MACHINE\Software\Policies\Microsoft Services\AdmPwd`, Data being a `1` for enabled or `0` for disabled. 

We should have what we need now. 

You want to create a new GPO for dynamic LAPS enablement. Just as before, you should be doing this initially in a test area of some kind. 

{% include callout.html content="I like to denote in the name of this policy that some elements within will be using Item-level targeting filters. I do this in my environment by putting **[ILT]** near the end of the GPO's name. I do this as a courtesy to other admins, and as a note to myself, that there's additional configuration within the policy that might prevent it from applying to computers or users that it's scoped to." type="primary" %}

{% include callout.html content="Feeling lazy? If you skim and understand these upcoming steps to create and configure the registry items, but would like to skip all the clicking, [I've included my policy's output below and you should be able to simply copy and paste it into your own policy](#xml-export-of-my-dynamic-laps-enablement-registry-items)[^fn-gppcopypaste]" type="success" %}

[^fn-gppcopypaste]: Group Policy Preferences supports XML-based copy/paste of most or all of its configuration items, and also of its Item-level targeting targets as well. I suggest you use this feature as much as possible, especially if you developing big policies with minor changes between each item, or you want to apply ILTs across several policies easily. It can really cut down on errors and tedium. It also generally supports right-click enabling or disabling of individual configuration items, which can help you out during testing and troubleshooting, and isn't as heavy-handed as having to enable or disable the entire policy. 

In the GPMC editor's windows of our policy, expand the left-hand page to `Computer Configuration` -> `Preferences` -> `Windows Settings` -> `Registry`. Right-click on `Registry` and click on  `New` -> `Registry Item`. Fill out the New Registry Properties as follows:

* Action: **`Update`**
* Hive: **`HKEY_LOCAL_MACHINE`**
* Key path: **SOFTWARE\Policies\Microsoft Services\AdmPwd**
* Value name: **AdmPwdEnabled** (don't click on the "Default" checkbox)
* Value type: `REG_DWORD`
* Value data: **1** (hex or decimal doesn't matter for this value)

![Image that shows the new registry item's properties filled out as just instructed](/assets/images/22-01-mdt-laps/misartg-LAPS-first-reg-entry-settings.png)

This creates the registry settings necessary to enable LAPS. 

Click on the `Common` tab to get to special GPP settings. **Click** the checkbox to enable `Item-level targeting` then **click** the `Targeting...` button to enter that menu.

![Image that shows the where to click to enable and configure Item-level targeting in the Common tab](/assets/images/22-01-mdt-laps/misartg-enable-then-configure-targeting.png)

In the ILT `Targeting Editor` menu, click on `New Item` then `Registry Match` to create a registry match targeting item. 

![Image that shows the where to click to create a new registry match ILT targeting item](/assets/images/22-01-mdt-laps/misartg-ILT-menu-new-regmatch.png)

Then configure your newly-created item as follows:

* `Match type`: Match value data
* `Value data match type`: Any
* `HIVE`: HKEY_LOCAL_MACHINE
* `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
* `Value name`: **AutoAdminLogon**
* `Value type`: Any
* `Value data`: **1**

![Image that shows where to input the initial config for the first registry match ILT item](/assets/images/22-01-mdt-laps/misartg-first-regmatch-item-initial-config.png)

What you've just done is create a check that will require the autologon setting `HKLM\SOFTWARE\Microsoft\Windows NT\Current Version\Winlogon\AutoAdminLogon` to be **1** for your policy to apply. **But this is the opposite of what we want**. Right-click your recently created item and choose `Item Options` -> **`Is not`** to reverse the sense of the check. 

![Image that shows where to click to reverse the sense of the ILT check with the Is Not selection](/assets/images/22-01-mdt-laps/misartg-first-regmatch-reverse-sense.png)

After you change to the **Is Not** sense, you also get some bonus logic like the registry value not existing in the first place, in addition to the values not matching. 

![Image that shows our negated ILT item check](/assets/images/22-01-mdt-laps/misartg-first-reg-match-now-reversed.png)

We want to create a second check, to verify that the autologon process is using the username of the MDT user. 

Click on `New Item` -> `Registry Match` and fill it out thusly:
* `Match type`: Match value data
* `Value data match type`: Any
* `HIVE`: HKEY_LOCAL_MACHINE
* `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
* `Value name`: **DefaultUserName**
* `Value type`: Any
* `Value data`: ***username of the local administrator account that MDT uses***. It's probably **Administrator**

Then right-click the item you just created and again choose `Item Options` -> **`Is Not`** and ensure that `Item Option` -> **`And`** is selected. 

![Image that shows both our ILT items for the LAPS enablement policy](/assets/images/22-01-mdt-laps/misartg-both-regmatch-items.png)

Click `OK` on the two windows to complete the configuration. 

{% include callout.html content="UPDATE - April 19, 2023: [See our updated guidance later in the page](#update-41923---updating-our-approach-for-windows-laps) given the [new Windows LAPS feature included with the April 2023 updates](https://techcommunity.microsoft.com/t5/windows-it-pro-blog/by-popular-demand-windows-laps-available-now/ba-p/3788747). We don't have screenshots of each setting, but if you've come this far, you can definitely follow the bullets. " type="primary" %}

You've now configured the base case/default policy, to enable LAPS if MDT (or some other software, potentially) is not configured to automatically log in to the computer with the Administrator user. 

We'll now create a second registry item to do the opposite: disable LAPS if the Administrator account is configured to autologon. 

{% include callout.html content="If you're comfortable with it, you could copy/paste your previous registry item, changing the value data and reversing the sense of the ILT checks, since that's pretty much all we're doing. If not, feel free to follow these steps below and you can double check against my screenshots if you wish." type="primary" %}

Click on  `New` -> `Registry Item`. Fill out the New Registry Properties as follows:

* Action: **`Update`**
* Hive: **`HKEY_LOCAL_MACHINE`**
* Key path: **SOFTWARE\Policies\Microsoft Services\AdmPwd**
* Value name: **AdmPwdEnabled** (don't click on the "Default" checkbox)
* Value type: `REG_DWORD`
* Value data: **0** (hex or decimal doesn't matter for this value)

Click on the `Common` tab, click the Item-level targeting checkbox, then enter the ILT menu by clicking the `Targeting...` box. In the `Targeting Editor` menu, click on `New Item` then `Registry Match` to create a registry match targeting item with the following settings:

* `Match type`: Match value data
* `Value data match type`: Any
* `HIVE`: HKEY_LOCAL_MACHINE
* `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
* `Value name`: **AutoAdminLogon**
* `Value type`: Any
* `Value data`: **1**

And again click on `New Item` -> `Registry Match` to create another ILT item:

* `Match type`: Match value data
* `Value data match type`: Any
* `HIVE`: HKEY_LOCAL_MACHINE
* `Key Path`: **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon**
* `Value name`: **DefaultUserName**
* `Value type`: Any
* `Value data`: ***username of the local administrator account that MDT uses***. It's probably **Administrator**

Like the ILT checks for the previous registry item, these should be in an **AND** relationship with one another, which is the default, and this time, these should be remain configured as **Is** since we want LAPS to be disabled if they're true. Click `OK` to close out of the ILT menu and new Registry item menu, saving your changes. 

There is a configurable **order** to the processing of Group Policy Preferences and Item-level targeting items. **With these, the lowest number applies first**[^fn-precedence-vs-gpp-order]. This is the order that we would want them in: the GPP item to enable LAPS goes first and would be overridden by item to disable it if that one matched instead. You shouldn't need to change the order, since you created them in the order you'd want, but know that you can if you had to for other GPPs/ILTs that you write. 

Feel free to compare your work to our screenshot of the second Registry item's configuration and its ILT configuration:

![Image that shows the second Registry item's settings and its ILT configuration](/assets/images/22-01-mdt-laps/misartg-second-regmatch-item-and-ilt-config.png)

#### Order your GPOs ####

In the `Group Policy Management` console, the `Linked Group Policy Objects` tab will show the order in which Group Policy Objects are applied in a given OU. The group policy lingo, GPO order is called **precedence**.

While not completely necessary, I'd recommend setting your 2 created GPOs such that the LAPS settings GPO applies first, and the dynamic enablement GPO applies after. I think this is a good idea because it should protect you someone else were to unwittingly enable/disable LAPS in the settings GPO; the dynamic enablement should apply after and "fix" things[^fn-precedence].

With GPO precedence, **the lowest number is applied last**. 

![Image that shows the second Registry item's settings and its ILT configuration](/assets/images/22-01-mdt-laps/misartg-GPO-precedence.png)

And you're pretty much done!

#### Test and check ####

With your settings in place, you can now check your clients.

On a client where you'd expect LAPS to be enabled, that is, a client that's not being actively built by MDT:

* Log into the test machine with an administrative account.
* Run `gupdate /force` to kick off an immediate group policy update process. This will lay down your LAPS settings, do the dynamic enablement, and if you're installing the LAPS client software via GPO, it will do that as well.
* Run `regedit` and navigate to `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft Services\AdmPwd`.
  * Check on the `AdmPwdEnabled` setting. It should be **`1`** to show LAPS is enabled. 
  * And you should see the other LAPS settings be applied as well, from the GPO using LAPS' ADMX. 

![Image that shows the registry with our LAPS settings, including LAPS being enabled](/assets/images/22-01-mdt-laps/misartg-verification-of-LAPS-enablement-via-registry.png)

* Run `gpupdate /force` again. With the LAPS settings in place, the LAPS client should be managing the password on this run. You can now use LAPS' tools to view the password and expiration date:
  * From an administrative console where you installed LAPS, you can use the `LAPS UI` application to see the LAPS password and its expiration date. *Note: If the previously set local admin password is younger than the `Password Age (days)` setting you use, LAPS won't reset it until it ages out*
  * Or, you can also use PowerShell to query a computer object's LAPS password and expiration time. Assuming the account you're using has permissions to read the attributes[^fn-attributereading], and [you have the `ActiveDirectory` PowerShell module installed](https://4sysops.com/wiki/how-to-install-the-powershell-active-directory-module/), the command `Get-ADComputer -Properties ms-MCS-AdmPwdExpirationTime,ms-MCS-AdmPwd -Identity <computername>` should do the trick. 
    * To convert the `ms-MCS-AdmPwdExpirationTime` attribute to something more human-readable, you can set it to a variable `$var` and run `[datetime]::FromFileTime([convert]::ToInt64($var,10)).DateTime`

[^fn-attributereading]: Some "in the weeds" notes regarding LAPS attribute permissions: It's considered best practice to have the list users who can read LAPS passwords be a small one, and have additional requirements around [when](https://www.beyondtrust.com/blog/entry/just-in-time-privileged-access-management-jit-pam-the-missing-piece-to-achieving-true-least-privilege-maximum-risk-reduction) or [from where](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/secure-administrative-workstations/ba-p/258424) they can read them. But the computer object also needs the ability to write new passwords and read+write password expiration into its computer object attributes. The computer will use the `NT AUTHORITY\SYSTEM` user to do this, and sometimes that can be another way to learn of a LAPS password's expiration datetime. The old-school way to run something as *SYSTEM* was to [wrap what you wanted to do in a Scheduled Task set to run as that user](https://superuser.com/a/604099), but you can also use [`psexec -s`](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) or the [`Invoke-CommandAs`](https://github.com/mkellerman/Invoke-CommandAs) PowerShell module as a bit of a shortcut. 

If that's all working, then your post-MDT/non-MDT clients should be good to go.

In terms of testing that this solved the original issue during MDT build scenarios, the *simplest* way to verify its effectiveness may be to just try some builds, particularly ones that were failing due to this issue, and see if they now work.

But the *most complete* way would be to edit one of your Task Sequences to include a [Suspend/Resume workflow](https://techcommunity.microsoft.com/t5/windows-blog-archive/mdt-2010-new-feature-3-suspend-and-resume-a-lite-touch-task/ba-p/706872). You can use suspend/resume to effectively pause a deployment to examine or fix something manually, then click the `Resume Task Sequence` icon on the desktop to continue the deployment. We make use of mulitple suspend/resume tasks on our thick image preparation sequences, and use the suspend/resume pattern while developing and troubleshooting complex sequences. It's a great, underutilized MDT feature! [Todd Lathome's blog has good instructions on how to create one](https://www.toddlamothe.com/deployment/pause-task-sequence-mdt-2010.htm) and an example of ours looks like this: 

![Image that shows a screenshot of an MDT Task Sequence that has a Run Command Line task to run LTISuspend.wsf](/assets/images/22-01-mdt-laps/misartg-mdt-suspend-task.png)

For testing this situation, you'd put your suspend/resume near the end of a Task Sequence, long after it should have joined the domain, and if you've performed reboots or done `gpupdate`s as part of your build, all the better. Then kick off a build/rebuild of a test system with that sequence.

When the build reaches the suspend, open up `regedit`, navigate to `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft Services\AdmPwd`, and ensure the `AdmPwdEnabled` value is **`0`**, indicating that LAPS is disabled. 

You can then click on the `Resume Task Sequence` desktop icon to resume the build, let it complete successfully, then try the checks above again to ensure that LAPS has become enabled. 

If that's all working, **you should be done**! Enjoy your builds working with the originally set password all the way through, and the rest having LAPS configured. 

---

### XML export of my dynamic LAPS enablement registry items ###

If you'd like to try pasting my configuration of the above 2 registry settings for dynamic LAPS enablement, complete with ILT configuration, my output is here:

{% gist 3dd8bf23aebed7d2924532b2990231d4 %}

You'd copy this XML, then right-click on your GPO's `Computer Configuration` -> `Preferences` -> `Windows Settings` -> `Registry` option in the left-hand pane of the Group Policy Management Editor, then click **Paste**[^fn-gppcopypaste]. 

![Image that shows where to paste my copied GPO settings](/assets/images/22-01-mdt-laps/misartg-LAPS-where-to-paste.png)

---

### Update 4/19/23 - Updating our approach for Windows LAPS ###
Microsoft [released a new version of LAPS](https://techcommunity.microsoft.com/t5/windows-it-pro-blog/by-popular-demand-windows-laps-available-now/ba-p/3788747), called ["Windows LAPS"](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview), on April 11th, 2023. It was included in patches released for Windows 10, Windows 11, and Windows Server 2019 and 2022[^fn-newlapsno2016server], so if you installed those updates, you now have Windows LAPS. 

[^fn-newlapsno2016server]: [Windows LAPS was does not work on Windows Server 2016](https://www.reddit.com/r/sysadmin/comments/12itqb9/windows_laps_available_today/jfvq5s3/?context=1), so if you're still using Windows Server 2016 and wish to manage LAPS on it, you'll still use the old method with the LAPS CSE and the old LAPS GPO settings. If you're moving forward with Windows LAPS in your environment, you may wish to [write a WMI filter to target your Windows Server 2016 member servers](http://www.nogeekleftbehind.com/2016/01/19/os-version-queries-for-wmi-filters/), and then restrict those old LAPS GPOs to those 2016 Server systems via your WMI filter, to manage both old and new LAPS in your environment. 

Luckily, the new Windows LAPS can manage clients running the older LAPS CSE in what they call Legacy Mode, so you can ease into Windows LAPS over time. Unfortunately, there is additional work to do in scenarios like the one we're try to solve here, preventing LAPS from applying during system deployment. But, fortunately, there's [now guidance from Microsoft on how to handle that](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-scenarios-legacy#disabling-legacy-microsoft-laps-emulation-mode), and we can integrate those changes into our existing GPP.[^fn-postwindowslapsdisableyourcseinstaller] 

Essentially, just as we disabled the old LAPS from applying in autologon situations, we now need to prevent the new Windows LAPS from applying in autologon situations as well. This can be done by defining a new registry value named `BackupDirectory` in `HKLM\Software\Microsoft\Windows\CurrentVersion\LAPS\Config` to have a `REG_DWORD` value of `0`, and triggering it to apply when autologon is active. We also need to delete this registry value when LAPS *should* apply.

I've already [updated our gist that contains our GPP Registry values, you could copy and paste](#xml-export-of-my-dynamic-laps-enablement-registry-items) this into your own GPO if you like. If you'd prefer to write them yourself, please add these two new GPP Registry items:

  * Create a new **Registry Item** to **Update** registry value `BackupDirectory` in `HKLM\Software\Microsoft\Windows\CurrentVersion\LAPS\Config` to `REG_DWORD` value of **`2`**, meaning Active Directory backup of the password. If you'd prefer to backup to Microsoft Entra instead, use a value of **`1`** .[^fn-thanksjan]
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
  * Create another new **Registry Item** to **Update** registry value `BackupDirectory` in`HKLM\Software\Microsoft\Windows\CurrentVersion\LAPS\Config` to a `REG_DWORD` value of **`0`**. 
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

[^fn-thanksjan]: This improvement over our original guidance is thanks to Jan Inge. Thanks for the suggestion, Jan! 

[^fn-postwindowslapsdisableyourcseinstaller]: It's not all good news, though. [Microsoft announced a regression that occurs when the old LAPS client-side extension (CSE) gets installed after the April 11th updates, which breaks both the old LAPS and the new Windows LAPS](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview#legacy-laps-interop-issues-with-the-april-11-2023-update). Thankfully, the resolutions are pretty straightforward, and hopefully they can find a way to fix the issue without needing intervention. 

![Image that shows our updated GPP Registry settings, and their order](/assets/images/22-01-mdt-laps/misartg-041923-dynamic-laps-gpo-settings-capture.png)
*Screenshot of our new GPP settings and their order*

It's possible we'll write a new version of this page that ignores the old/legacy LAPS and solely targets the new Windows LAPS, but for now you might have to worry about both the old and new LAPS in the same environment just as we do, and this new guidance will work for you with both.


---
### Footnotes ###
