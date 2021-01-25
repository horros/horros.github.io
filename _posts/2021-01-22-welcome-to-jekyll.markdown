---
layout: post
title:  "Disabling rules in ModSecurity"
date:   2021-01-22 00:42:19 +0200
categories: jekyll update
---

[ModSecurity](https://modsecurity.org) is an Open Source Web Application Firewall.
It provides real-time monitoring, logging and access control for your web apps by 
inspecting incoming traffic and detecting anomalies. It can also provide complete 
HTTP traffic logging (web servers are by default really bad at logging anything 
that's of any use when things go wrong), it can help you harden your web apps by
enforcing rules and restrictions, and can help you fix cross-site request forgery
(CSRF) vulnerabilities. 

OWASP (the Open Web Application Security Project) provides a daunting set of rules for
ModSecurity that are a great starting point for your WAF. There are, however, cases
when you need to disable certain rules for certain web applications. In some cases,
some of our customers send a `Content-Type`-header even for HTTP GET requests 
without any body content. This is normally not a big deal, but the OWASP ModSecurity
Core Rule Set (CRS) includes by default rules that forbid `Content-Type` headers
for HTTP GET or HEAD requests.  
As the `Content-Type`-header tells the recipient "the stuff I'm sending you now 
is this kind of content (like a PDF or XML or JSON data)" so that the recipient can 
figure out how to handle it, this should *not* be sent if there is no data actually 
being sent. The CRS also forbids sending body content in HTTP GET requests altogether 
(this is a good thing, as this *may* allow users to — accidentally or otherwise — 
overwrite request parameters using the body of the request).  
Normally to disable certain ModSecurity rules, you can just modify your main 
ModSecurity configuration file and add the line `SecRuleRemoveById <rule_id>` 
(eg. `SecRuleRemoveById 950002`). Having a look at the default main ModSecurity
configuration file, there are two (well, three) rules that are of interest to us:

However, instead of disabling rules system-wide,
we can disable them based on criteria like request URI or query parameters.

To disable the `Content-Type`-header checking for only the URL `/api`, we'll 
do the following:

Depending on where you've installed the CRS (I'm going to use 
`/etc/nginx/modsec/coreruleset-3.0.0` as an example here), navigate to the `rules`
folder. In this folder you should see a file called 
`REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example`. In this file you can define 
rules that should be applied *before* ModSecurity executes (there should also be a file
called `RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example` in which you can define
rules that are applied *after* ModSecurity has run). Remove the `.example` extension 
from the file and open it up.



