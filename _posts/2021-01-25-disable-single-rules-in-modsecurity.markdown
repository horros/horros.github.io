---
layout: post
title:  "Disabling single rules in ModSecurity"
date:   2021-01-25 22:40:19 +0200
comments: true
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

The `Content-Type`-header tells the recipient "the stuff I'm sending you now
is this kind of content (like a PDF or XML or JSON data)" so that the recipient can
figure out how to handle it. Therefore the header should *not* be sent if there is
no data actually being sent in the body. The CRS also forbids sending body content
in HTTP GET requests altogether (this is a good thing, as this *may* allow users to
— accidentally or otherwise — overwrite request parameters using the body of the request).

Normally to disable certain ModSecurity rules, you can just modify your main
ModSecurity configuration file and add the line `SecRuleRemoveById <rule_id>`
(eg. `SecRuleRemoveById 950002`). The default install of ModSecurity should have
a rule with ID 200000 in the `modsecurity.conf`-file which is of interest to us in this case.

    SecRule REQUEST_HEADERS:Content-Type "(?:application(?:/soap\+|/)|text/)xml" "id:'200000',phase:1,t:none,t:lowercase,pass,nolog,ctl:requestBodyProcessor=XML"

Let's have a quick look at the rule and what it does. The basic syntax for a SecRule is `SecRule <variable(s)> "<operator>" "<transformations>"`

What the rule says is _In the header parsing phase (phase:1), remove all previous
transformations of the value (if any) and convert it to lowercase (this is t:none,t:lowercase).
If the lowercase content of the `Content-Type` request header contains either application/soap+xml
or text/xml, allow the request to continue (this is pass), don't log the rule, and run the body
content through the XML Body Processor_.

_For more information about the SecRule syntax and all the variables and transformations and actions and so on, have a look at [The reference manual](https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual-%28v2.x%29) (there is currently no reference manual for version 3.x)._

What we *could* do is add a line to the end of the `modsecurity.conf` line:

    SecRuleRemoveById 200000

and we would accomplish our task.  However, instead of disabling rules system-wide, we can
disable them based on criteria like request URI or query parameters.

To disable the `Content-Type`-header checking only for the URL `/api`, we'll do the
following:

Because we are disabling rules, the rule that disables other rules must be defined
before the rule it's disabling. In our case, we need to define the rule in `modsecurity.conf`,
before the line with the rule id 200000. We'll add this line:

    SecRule REQUEST_URI "@beginsWith /api" "id:1001,phase:1,t:none,t:lowercase,pass,log,msg:'Don't parse XML body for /api URIs',chain" \
        SecRule REQUEST_HEADERS:Content-Type "(?:application(?:/soap\+|/)|text/)xml" "t:none,t:lowercase,chain" \
        SecRule REQUEST_METHOD "^GET$" "ctl:ruleRemoveById=200000"

To explain the rule:

- Create a rule with ID 1001 that runs in the header parsing phase (phase 1)
- Remove any possible previous tranformators, and transform the REQUEST_URI value to lowercase
- Check if this value begins with "/api"
- If so, allow the request to continue, log a message, and chain with the following rule:
    - If the lowercase value of the Content-Type request header contains either `application/soap+xml` or `text/xml`, chain with the following rule
        - If the REQUEST_METHOD value is "GET", disable rule with ID 200000

Save, reload the Nginx configuration and lo and behold, ModSecurity no longer blocks
GET requests to URLs where the path starts with `/api` if the request header includes `Content-Type: text/xml`!

As a final note, the main `modsecurity.conf`-file is not the optimal place for your custom rules.
In this case it was necessary because of the requirement that rules that remove other rules must
be defined before the rules they remove. However, for _most_ cases, the correct place to add your
custom rules is in the Core Rule Set.

Depending on where you've installed the CRS (I'm going to use
`/etc/nginx/modsec/coreruleset-3.0.0` as an example here), navigate to the `rules`
folder. In this folder you should see a file called
`REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example`. In this file you can define
rules that should be applied *before* ModSecurity executes. There should also be a file
called `RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example` in which you can define
rules that are applied *after* ModSecurity has run. Remove the `.example` extension
from the files, edit them, and reload the web server configuration and you should be set!

/ML

<div id="disqus_thread"></div>
<script>

    var disqus_config = function () {
    this.page.url = "{{site.url}}{{page.url}}";
    this.page.identifier = "{{page.id}}";
    };
    
    (function() { // DON'T EDIT BELOW THIS LINE
    var d = document, s = d.createElement('script');
    s.src = 'https://markusblog-2.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>