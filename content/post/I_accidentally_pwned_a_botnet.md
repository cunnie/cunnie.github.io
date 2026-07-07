---
title: Why Did AWS Charge Me $122.93 This Month?
date: 2026-07-02T11:07:27-07:00
draft: true
---

## tl;dr

My website was being hammered by clients

## The Surprise Bill from AWS

I have a dinky little webserver on AWS [0] that costs **$12.11** a month. Until today, when I got June's invoice.

From: Amazon Web Services <invoicing@aws.com>

```
 Service Charge Details
----------------
AWS Data Transfer --- $124.90
Amazon Elastic Compute Cloud --- $10.66
Amazon Virtual Private Cloud --- $3.60
```

**$124.90** in AWS Data Transfer!? What's going on? The Amazon email has a helpful link, "your account history are available on the [Billing & Cost Management page](https://console.aws.amazon.com/billing/home#/bills?year=2026&month=6)." I expand the _Data Transfer_ section, and, under that, the _US East (N. Virginia)_ section. It's mostly $0, except for the last item:

> $0.090 per GB - first 10 TB / month data transfer out beyond the global free tier
	
1,365.874 GB
	
USD 122.93



I ssh onto the box: `ssh blocked.sslip.io`. The disk is full.


## Footnotes

[0] [t4g.micro](https://aws.amazon.com/ec2/instance-types/t4/), 2 Arm cores 1GB RAM