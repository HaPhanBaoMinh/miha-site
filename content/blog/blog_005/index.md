---
title: "AWS Assume Role vs AWS Access Keys credentials"
description: "Understand the difference between using long-term Access Keys and temporary Assume Role credentials in AWS, and what case to use each for secure access management."
author: "MinhBoo"
date: 2025-10-26T10:00:00+07:00
draft: false
tags: ["aws", "iam", "sts", "devops", "security", "eks"]
categories: ["Cloud", "DevOps Guides"]
hideSummary: false
summary: "This article compares AWS Access Keys and Assume Role (STS) — explaining how they work, their pros and cons, and why Assume Role is the future for secure cross-account access and service identity management."
---

## 1. Why should you care?

If you’ve ever store AWS Access Key into your CI/CD pipeline, `.env`, or `Kubernetes Secret` — and then worried about it being leaked, misused and you need to spend time rotating it regularly…

That’s the problem.

As organizations move toward **multi-account, short-lived credentials, and zero-trust**, AWS *Assume Role* becomes a more secure and manageable way to grant access to AWS resources.  

In this article, we will discuss about:
- IAM Users, Roles, and Policies.  
- How Access Keys and Assume Role differ in authentication flow.  
- Practical examples using AWS CLI and EKS Service Accounts (IRSA).  
- Pros, cons, and real-world trade-offs for each approach.  

---

## 2. IAM Users, Roles, and Policies Overview 

![blog_005](images/01.jpeg)

