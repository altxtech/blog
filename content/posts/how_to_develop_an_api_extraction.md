---
title: "Shorter title"
date: 2023-08-19:00:00-03:00
draft: true
description: Will this fix it?
---
## Intro

The task is very simple. You want to extract data from and API and load it into something like block storage,
or a database.  
On the surface level, it sounds like the type of thing that *should* be simple. And it is... for smaller extraction jobs
that take a few minutes at most to complete. However, when it comes to larger extraction jobs (hours to complete), and
making it a resilient, reliable, production-ready solution, it gets suprisingly complicated very fast.

### What makes a data extraction program production ready?

**At a minimum**, for a production environment, a data extraction should achieve at least these 3 things:
1. **Recover from failures.** For an hours-long job, it is *not viable* to restart it from scratch everytime there is an unforseen edge case exception or a random system outtage. Also, your program should be able to anticipate and handle some basic errors.
2. **Easy to debug.** On a large enough job, the law of large numbers clearly states that errors are *guaranteed* to happend. There is no going around that, but you should be able to know when and why the extraction failed, and possibly estimate an SLA for the extraction (is it 99% accurate? Or 99.9999%? You should be able to tell).
3. **Retry failed requests**. If you know which requests failed and why, you should then be able to fix your code and retry those, making your extraction as reliable as possible.  

**Idempotency is desirable, but not required**, which means that running a part of the extraction multiple times will
have the same end result as running it only once on wherever you're storing your data. In the data world, in practice,
not having idempotency will mean having duplicate data in your destination. You can handle deduplication downstream, but itis nice having it handled at the extraction step. Besides, making your replicated data a 1-for-1 to the source, opens some
new alternatives to auditing the result.  

**Do not worry about performance initially**. It doesn't matter if the extraction takes long as long as it is reliable.  
Your engineering hours are more valuables than some VM-hours. So, don't feel tempted to mess with asyncio if the
extraction is feasible without it. Would only be worth it if would save several days of runtime OR if you want to run 
the job multiple times. 

### Should you even build it?

Before you start coding, it is worth mentioning that maybe you could just... not. For most popular SaaS applications,
there are pre-built connectors offered by ETL/ELT platforms such as [Fivetran](https://www.fivetran.com/), 
[Stitch](https://www.stitchdata.com/), [Airbyte](https://airbyte.com/), among others.  

These platforms often offer generous free tiers, and even if you have to pay, in practice the cost of these platforms
will be less than the opportunitty cost of your engineering hours.  

If you are building an data migration or data connector for the API of a popular service and you were unfamiliar with
the products mentioned above, you should definitely take a look at those first. 

With that said...  

You can't always find a suitable connection for your business needs. Maybe because we're talking about a smaller,
unpopular, niche platform, or because the connectors offered by vendors do not meet some business required.  

In those cases, we are going to need to code it ourselves. So let's get to it.

## Coding from scratch
### Level 1 - State
### How to deploy?
### Level 2 - Making it faster with asynchronous programming
### Level 2.1 - asdf
### Level 4 - Using message brokers
### Level 5 - Using orchestrators

## Custom connector on an ELT platform
### Fivetran
### Rudderstack
### Option 3
### Option 4
