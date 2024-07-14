+++
title = 'Change Data Capture, Debezium and dangerous database cleanups'
date = 2024-07-14T12:34:55+01:00
draft = false
+++
## Introduction

I have been wondering for a while what my second blog post's topic would be about. Where would I get inspiration from?

And then it dawned on me...

I have an actual day job I can use for inspiration!
![captain obvious](/images/captain_obvious.gif)

I am going to write about the process of archiving data from a relational database in which I have enabled Change Data Capture through Debezium.

## Context

Lately I have been developing a full stack application which is powered by a relational database. One of the hard requirements
of the application was to visualise the business metrics on a dashboard **in real time**. 

If you ask any stakeholder how fresh they want their data they will 99.99% answer real-time fresh. Although this has become 
an actual meme in the real world, in my case I could not avoid it as the application itself is time critical and the fresher the data 
the more errors the end user would avoid.

But if I have one advice for other engineers, this would be to **always challenge a request for real-time data**
![real time meme](/images/real_time_meme.jpeg)

## The problem

Moving past the funny memes, I had to come up with an architecture that would provide both dashboards with real time data
and also somehow take the burden off from the application's operational database. We don't want to perform CRUD database operations
and at the same time calculate aggregations over the same tables. That would slow things down for both operations
and the dashboards users in the long run.

## The proposed solution

Although this problem can be probably (or partially) solved by using an RDS cluster (e.g. AWS aurora) that will have write and read replicas,
I preferred to use Kafka as a middleware instead


Enter Change Data Capture: this is a high level design of what I came up with
[![cdc](/images/cdc.png)](/images/cdc.png)

To be continued...