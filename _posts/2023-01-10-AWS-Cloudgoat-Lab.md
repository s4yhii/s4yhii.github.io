---
title: Cloudgoat rce_web_app scenario
date: 2023-01-10 12:00:00 -0400
categories: [Web Security, AWS]
tags: [web security, labs, owasp, rce, cloud, aws]
---

# Cloudgoat RCE_WEB_APP Scenario

## Introduction
CloudGoat is a training and learning platform developed by Rhino Security Labs to help individuals and organizations understand the risks and vulnerabilities associated with cloud-based applications. One of the scenarios available on CloudGoat is the RCE_web_app scenario, which allows users to practice exploiting remote code execution vulnerabilities in a web application running on the cloud.

In this blog post, we will walk through the RCE_web_app scenario in CloudGoat and provide a step-by-step guide on how to exploit the vulnerability and gain access to the application's backend. We will also discuss the significance of this vulnerability and how it can be prevented in real-world scenarios. By the end of this post, you should have a better understanding of the risks and challenges associated with web application security in the cloud and how to mitigate them. So, let's get started!

## Solution 1

When deploying the laboratory we have access to 2 users: Lara and MCduck, first we will start listing the services with the user Lara.

```bash
aws configure --profile Lara
```

### Finding EC2 Instances
We decided to start by finding out which EC2 instances Lara has access to by running the following command.

```bash
aws ec2 describe-instances --profile Lara
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/1.jpg)

The output shows that Lara has access to an EC2 instance with a public IP. However, when we try to navigate to the IP in a browser, we get a timeout, so I decided to look at load balancers with this command

### Finding Elastic Load Balancers

```bash
aws elbv2 describe-load-balancers --profile Lara
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/2.jpg)

We see a publicly accessible load balancer, we have the public DNS name, so we access it from browser and we see a landing page with nothing interesting, so if it exists a webpage, it needs a bucket to stores all the static files.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/3.jpg)

### Finding S3 Buckets

```bash
aws s3 ls --profile Lara
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/4.jpg)

The output of this command shows us that Lara can list 3 buckets, as pictured below.

Then we need to list the content of buckets with this command:

```bash
aws s3 ls s3://<bucket> --recursive --profile Lara
```
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/5.jpg)

This bucket appears to contain logs for the load balancer we discovered earlier, so we download the content with this command.

```bash
aws s3 cp s3://cg-logs-s3-bucket-rce-web-app-cgidtjk4gmqpko/cg-lb-logs/AWSLogs/261824994497/elasticloadbalancing/us-east-1/2019/06/19/555555555555_elasticloadbalancing_us-east-1_app.cg-lb-cgidp347lhz47g.d36d4f13b73c2fe7_20190618T2140Z_10.10.10.100_5m9btchz.log . --profile Lara
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/6.jpg)

We see an URL in the log files, after accessing to the URL we got a timeout, this path 'mkja1xijqf0abo1h9glg.html' keeps repeating in all log urls, so maybe we need a valid url to access it, so we append the path to the previous URL we've found and we got a page to run commands.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/7.jpg)

### Getting Access via SSH

From the EC2 enumeration we know that the instance has ssh enabled, but EC2 instances uses keys not credentials.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/7.jpg)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/8.jpg)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/AWS/rce_web_app/9.jpg)
