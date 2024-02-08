---
layout: post
title:  "PDQ technique: Remediation deployments"
author: John Moeller
date:   2024-02-08 10:10:10 -0700
tags:   [pdq, pdqdeploy, pdqinventory, deployment, installs, remediation, automation]
---

### Background ###

We are big fans of [PDQ](https://www.pdq.com)'s products, starting with Deploy many years ago and a few years back adding Inventory so we could move to [(legacy) LAPS integrations for our deployments](https://help.pdq.com/hc/en-us/articles/115001132352-LAPS-Integration-with-PDQ-Inventory-and-PDQ-Deploy), among other benefits. Their reliability, simplicity and features are a pleasant surprise, and we've been able to use the software to effectively solve problems and make our own lives easier. And the pricing model works really well for our small team. 

As I've talked to more colleagues about how we use the software, it became obvious that we needed to write up a particular way in which we use these products. Not because our technique is novel or even particularly interesting, as I'm sure most folks are doing something similar with their PDQ environments. But it is a little complex and perhaps not an obvious way to use the products for beginners, who may not realize that PDQ D&I can be used for hands-off, automated deployments or configuration management jobs. We've come to call the technique "Remediation Deployments" internally. 

## Our approach: Remediation Deployments ##

![Image showing a cycle diagram of our Remediation Deployments workflow](assets/images/24-02-pdq-remediation/misartg-pdq-remediation-cycle-diagram.png)

The idea is simple:

1. Create or update your PDQ Inventory Scan Profiles/Scanners to detect systems that are out of compliance with your desired state. Be sure to set Triggers that run often enough to meet your time resolution goals, without running excessively. 

2. Create a PDQ Inventory group that dynamically contains systems that lack the desired state. This is the "Remediation Group". You may not desire every system to have this particular state or config, so scope this group to only the audience of *potential* machines you'd want.

[^fn-temporarilyunavailablecomputers]: We take this a step further in our environment by having group structures that detect temporarily unavailable machines, whether that be because they're offline, unmanageable or broken, non-compatible OSes (say, Linux), or what have you. We then exclude the temporarily unavailable machines from nearly every group that's scoped by a deployment. Like other dynamic groups, temporarily unavailable computers will leave those groups once they become available to manage again. We prefer this approach to seeing dozens of errored clients in our deployments, but it's mostly a style preference. 

3. Optional: Create a related PDQ Inventory group that identifies systems with the state or configuration that you desire. It isn't strictly necessary to do this, but you may want to, as it could help you with the other steps, analysis/reporting, or just for completeness. If you do create a group like this, you can use it to verify that there's no overlap in membership with your remediation group. 

4. Create a PDQ Deploy package that modifies a system, changing its state or configuration to something you desire. This can be as simple as an application install or update, or something less structured/more complicated.

5. Create a recurring Schedule in PDQ Deploy to deploy the package to the Remediation Group; this is the "Remediation Deployment". Our Remediation Deployments generally run hourly, some more frequently than that.

When working correctly, the machines that don't have what you want will end up in the Remediation Group, and be remediated by the Remediation Deployment, without you having to do anything. Then they fall out of the Remediation Group as a result of your remediating PDQ Deploy package, PDQ Inventory scans, and group membership rules.

### Remediation Deployment concept example ###

Let's walk through the concept with an example. 







