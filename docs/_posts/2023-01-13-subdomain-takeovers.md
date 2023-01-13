---
layout: post
title: "Hunting for subdomain takeover vulnerabilities"
date: 2023-1-13 00:00:00 -0000
categories:
tags: [bugbounty,vdp,subfinder,subjack]
---

## Background
I've recently had some success on bug bounty and vulnerability disclosure programmes with finding subdomains that are vulnerable to takeover attacks. These types of attacks are very common and require good processes to prevent them from occurring, so I thought I would use this post to document how I've been finding vulnerable subdomains, along with some recommendations for preventing them. Before we get to that though, lets get the terminology clear by asking...

## What is a subdomain takeover?
Subdomain takeover attacks are a type of security vulnerability that occurs when an attacker is able to claim ownership of a subdomain that is no longer in use or actively maintained by the organisation that originally registered it. This can happen if the organisation has abandoned the subdomain, or if the DNS records for the subdomain have been configured incorrectly.

## How do you find vulnerable subdomains?
My preferred way to hunt for vulnerably subdomains is to use a combination of two open-source tools: [Subfinder](https://github.com/projectdiscovery/subfinder) and [Subjack](https://github.com/haccer/subjack). Subfinder is a subdomain discovery tool that can be used to identify subdomains by using various sources such as certificate transparency logs, search engines, and others. Subjack, on the other hand, is a subdomain takeover tool that is used to check if a subdomain is vulnerable to takeover by sending a request to the server and checking the response.

The basic process involves first running Subfinder to identify a list of target subdomains, then running Subjack against each subdomain in the list to check if it is vulnerable to takeover. If your on the Blue team for an organisation and only need to monitor a fairly static list of domains then that's about as far as you need to take things. For bug bounty hunting though, the key is to automate the subdomain enumeration and testing process as much as possible so that you can handle the thousands of domains that are in scope across the various programmes.

## How can you prevent subdomains from becoming vulnerable?
Preventing subdomain takeover attacks is an important aspect of maintaining a secure online presence. Here are a few recommendations for organisations to prevent these attacks:

1. Keep track of all subdomains that are registered and actively maintain them.

2. Regularly review the DNS records for all subdomains to ensure they are configured correctly. You should pay particular attention to CNAME records that point to services that are easy to execute takeover attacks on (e.g. AWS S3).

3. Use a subdomain monitoring service to be notified of any changes to the subdomains.

4. Implement a process for reporting and addressing subdomain takeover vulnerabilities as soon as they are discovered.

5. Regularly perform security testing for all subdomains to identify vulnerabilities that could be exploited in a takeover attack.

By using the combination of Subfinder and Subjack, and following the above recommendations, organisations can proactively hunt for and remediate takeover vulnerabilites before they can be exploited.