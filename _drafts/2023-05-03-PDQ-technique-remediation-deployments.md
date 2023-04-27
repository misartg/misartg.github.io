---
layout: post
title:  "PDQ technique: Remediation deployments"
author: John Moeller
date:   2023-05-03 10:10:10 -0700
tags:   [pdq, pdqdeploy, pdqinventory, deployment, installs, remediation, automation]
---

### Background ###

We are big fans of [PDQ](https://www.pdq.com)'s products, starting with Deploy many years ago and a few years back adding Inventory so we could move to [(legacy) LAPS integrations for our deployments](https://help.pdq.com/hc/en-us/articles/115001132352-LAPS-Integration-with-PDQ-Inventory-and-PDQ-Deploy), among other benefits. Their reliability, simplicity and features are a pleasant surprise, and we've been able to use the software to effectively solve problems and make our own lives easier. And the pricing model works really well for our small team. 

As I've talked to more colleagues about how we use the software, it became obvious that we needed to write up a particular way in which we use these products. Not because our technique is novel or even particularly interesting, but because it's a little complex and perhaps not an obvious way to use the tools for those new to them. But we've found it a worthwhile way to automate some of our common tasks, and as we delved into some more advanced PDQ techniques, beyond just application installations and updates, we've used it more and more. We've come to call the process "Remediation Deployments" internally. 

## Our approach: PDQ Remediation Deployments ##

The basic idea is simple:

1. Create a PDQ Deploy package that modifies a system, changing its state or configuration to something you desire. This can be as simple as an application install or update, or something less structured/more complicated. 
2. Create a PDQ Inventory group that identifies systems with the state or configuration that you desire. It may not be necessary to do this, but you may want to, as it could help you with the other steps, analysis/reporting, or just for completeness. 
3. Create a related PDQ Inventory group that identifies in-scope systems that lack the desired state. This is the "Remediation Group". You may not desire every system to have this particular state or config, so scope this group to only the audience of *potential* machines you'd want. 
4. Ensure the groups in PDQ Inventory are accurately portrayed over time. You may need to schedule or tweak your Scan Profiles to ensure the membership is up-to-date and accurate.
5. Create a recurring Schedule in PDQ Deploy to deploy the package to the Remediation Group; this is the "Remediation Deployment". Our Remediation Deployments generally run hourly.

If you do it right, the machines that don't have what you want will end up in the Remediation Group, and be remediated by the Remediation Deployment, without you having to do anything.

**create a graphic and insert it here**

### Remediation Deployment concept example ###

Let's walk through the 5 steps with an example. 





