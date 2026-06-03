---
layout: content.njk
contentType: project
title: Identifying Health Checks
description: A framework to prioritize customer health checks.
date: 2026-01-15
tags: [automation, duct tape special]
hero: ./src/assets/img/project/anchor/hero.png
heroAlt: A slide presentation describing ANCHOR
list: yes
---
# Overview
The goal in creating a portal to identify pending, upcoming, and overdue health check targets for customer accounts is a challenge some account teams struggle with. Secondly, depending on data consistency & hygiene, technical seller influenced incremental revenue is sometimes difficult to identify. 

This is a solvable problem: create a roadmap for technical sellers to identify their top targets for health check ready accounts, ensure there is follow-through on the logging so the correct visibility and credit is assigned.
<br>
<br>

## 3 Questions Framework
1. What is it?
- A landing page to correlate account data and activity data to identify accounts with upcoming health checks
2. Why does it matter?
- If there are many accounts in your territory, it becomes difficult to prioritize or find the right opportunities to pursue
3. How does it work?
- Via API pulls custom reports via $CRM$ (3 .csv's), then analysis is done (event IDs of reports correlated to timestamps of activities on accounts, to account owners). From there a webpage is rendered with the analyzed data to allow users to identify their targets

<br>
<br>

## Problem Statement
Health checks are a critical lever for finding expansion opportunity of account teams, and pre-empty customer satisfaction issues with mis-deployments. The current process is a mix of pulling data from CRM tools, data lake reports, backups from customers, and then manual presentation creation. Although there are some standardized tools, delivery to customers varies greatly. 

Some known challenges: no automation, very difficult to identify pipeline generated from HC, timelines of when to think & deliver nonstandard, and difficult for team management to assess efficacy of the delivery / completion. 

Expected output: Discovery call from technical leadership for requirements gathering, deeper understanding of the need occurs 1 week before the summit. The summit is a 3-day hackathon-like event to generate an action-oriented document (no more than 10 slides) documenting specific recommendations and actions leadership need to take. What tools might be required to complete the actions, and feedback for the next cohort. 
<br>
<br>

# Solution
<br>

## $CRM Reports
3 reports are required. The tool does not allow for native analysis between these reports as they are from different views within the application and nobody on this team possessed the correct visibility to create a worldwide view. 
<br>
<br>

### Activities
|  Name  |  Account | Opportunity | Subject       | Revenue | Timestamp         | Attachment |
|:------:|:--------:|:-----------:|:-------------:|:-------:|:-----------------:|:----------:|
| Nic W  | ACME ORG | Refresh     | Architecture  | $MM     | YYYY-MM-DD-HH:MM  | N          |
| Nic W  | EXAMPLE  | Renewal     | Support       | $M      | YYYY-MM-DD-HH:MM  | N          |
| Nic W  | Company3 | New Logo    | Demo          | $K      | YYYY-MM-DD-HH:MM  | N          |

<br>

### Accounts
|  Owner  |  Account | GEO   | Region   |
|:-------:|:--------:|:-----:|:--------:|
| Nic W   | ACME ORG | AMS   | Bay Area |
| Nic W   | EXAMPLE  | MEA   | Central  |
| Nic W   | Company3 | JAPAC | Japan    |

<br>

### Contracts
|  Account | Product | Start Date    | End Date    | Amount |
|:--------:|:-------:|:-------------:|:-----------:|:------:|
| ACME ORG | VM      | YYYY-MM-DD    | YYYY-MM-DD  | $M     |
| EXAMPLE  | SaaS    | YYYY-MM-DD    | YYYY-MM-DD  | $K     |
| Company3 | HW      | YYYY-MM-DD    | YYYY-MM-DD  | $1     |

<br>

## Correlation
From here, bubbling up the correct accounts and reports became a matter of policy: at what date before a contract ending or after a contract starting *should* a HC be conducted? How many accounts possess contracts within that date window (at the time the flask app compiles)? Of those accounts, which posses an activity with subject = Health Check AND attachment = Y in that same window?

The remainder is the list of accounts which are HC candidates. Now a web portal is able to be built which can filter by account owner, geo/region (manager visibility), and flags are set for past due (delivery date occurs in the past per policy), pending (soon), or upcoming (future quarters). Sort by $amount from contracts report. 
<br>
<br>

## Deployment
A more difficult question is where would this app live? Typically IT is required to deploy a server and app, and IT Security will certainly wish to evaluate the application. Ultimately, they took over the app and ensured SAML/SSO adhered to RBAC and migrated from a VM one of our members operated to a full severless / public cloud app. 
<br>
<br>

## Iterations
<br>

### v1
Originally, we didn't possess the correct permissions to download SFDC reports via API. Therefore, we hit it with a hammer. Selenium to harvest a browser token, impersonate the user in a browser window, and click specific coordinates to download the appropriate report. Highly fragile.
{% image "src/assets/img/project/anchor/collection.png", "A screenshot of downloading the code", "prose-img-lg" %}
{% image "src/assets/img/project/anchor/processing.png", "A screenshot of processing the data", "prose-img-lg" %}
{% image "src/assets/img/project/anchor/reporting.png", "A screenshot of the data reporting", "prose-img-lg" %}
<br>

### v2 (alpha)
During the hackathon, a few friends in different organizations took pity upon us and provided us a few API keys, but not $CRM. This allowed the setup to migrate into PowerAutomate to download the reports to a specific repository which a web app could reach. Undeterred from a lack of direct access, with a few API keys we were able to then subscribe a new inbox to the reports as they were generated to start the automation. 
{% image "src/assets/img/project/anchor/powerautomate.png", "A screenshot of the powerautomate flow", "prose-img-lg" %}
A significant update is then paved, a dynamic web portal can hit an internal repository much easier than local scripts / reports downloading for each technical seller to parse.
{% image "src/assets/img/project/anchor/alpha.png", "A screenshot of the alpha UI", "prose-img-lg" %}
<br>

### Production (beta)
With a functioning web app, it underwent internal audit to receive the sign offs from IT, Security, and other groups. 
{% image "src/assets/img/project/anchor/beta.png", "A screenshot of the production (beta) UI", "prose-img-lg" %}
<br>

## Outputs
Here's my favorite quote from our action document, at the top in bold: "The Hawkins team will not be accepting feature requests or improvement feedback; it is delivered as is until more days are allotted for project scope changes and revisions."

The outcomes are even better when a professional engineer, or group of them, take the ideas and enterprise-ready them. 😃
{% image "src/assets/img/project/anchor/leaderboard.png", "A screenshot of the production UI", "prose-img-lg" %}

### Data Hygiene
We all learned something about our $CRM from this project. For example, Customer_Until date is the latest expiring contract on an account, so indexing all opportunities is required. This created the need for a major rewrite during v1 delivery.
<br>
<br>

# Appendix

<br>

## Notes
This project included a couple sleepless nights during the hackathon, and plenty of not-fun debugging. Using accrued political capital from the group to beg, borrow, and steal resources to deliver a high-visibility win for senior technical leadership. 

IT ended up picking this project up because they agreed with the software authors: if we possess tools that absorbed PIANO that are sourced from the data lake, it is possible the slide creation as well as target accounts etc all happens in an automated fashion at a time before the health check is due. Human intervention in the loop occurs when customers are not enrolled in the data lake. 

We estimate this tool saved over 10,000 hours annually for world wide field operations.