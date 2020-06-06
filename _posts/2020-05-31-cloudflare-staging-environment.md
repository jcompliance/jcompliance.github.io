---
layout: post
title:  "Setting up Cloudflare Staging Test Environment"
ref:  20200531a
date:   2020-05-31 15:00:00 +0800
categories: cloudflare onboarding
tags: cloudflare configuration
lang: en
---

Not really a thorough tutorial but a quick guide for administrators who require so-called `Staging Environment` where you can test your new CDN or security configuration in a sandbox environment. Like:

- I want to deploy a new Firewall Rule (or a caching rule, vice versa) and want to verify first with my QA team if my legit app request will remain safe.
- I want to deploy a new Cloudflare product but want to test it with my staging env before deploying it to my production.
- And many others.

As far as I know (and wish) that's a equivelant work going on by Cloudflare's product team to provide this. But if we need it today, there's a workaround. So this article will walk you through on how I guide my customers to set one up. 

# Steps
{:.no_toc}

* TOC
{:toc}

# (Optional) Set up a dedicated dashboard for staging domain name

This is optional step but it will be more handy to have another separated dashboard for only staging subdomain where you can freely play with. 

Decide a domain for staging environment. In this example, I'll be using `staging.blog.jeann.net` for staging environment of `blog.jeann.net`. Using subdomain is handy but having subdomain-level dashboard is ENT only feature. If you're not an enterprise customer, it's ok to set up a staging domain name under different root domain.

So, let's say your production domain is `blog.jeann.net`

- I want to use `staging.blog.jeann.net` as staging domain name and separate a dedicated dashboard for this host.
 -> This is enterprise only. [More read here](https://support.cloudflare.com/hc/en-us/articles/360026440252-Understanding-Subdomain-Support)
- I would rather use different root domain like `blog.jeann.cf` to set this up.
 -> Possible for everyone. Just follow usual onboarding steps.

You may proceed to `add a site` button at the Cloudflare dashboard to finalize this step.

# Add a new subdomain in Cloudflare DNS

I have a DNS entry in my dashboard that connects `blog.jeann.net` to the origin instance, which looks like below.

```
| Type         | Name                   | Content          | Proxy Status |
| A/AAAA/CNAME | blog.jeann.net         | <ORIGIN ADDRESS> | ON           |
```

Go to your new dashboard and make sure your staing domain has same origin address with proxy turned on.

```
| Type         | Name                   | Content          | Proxy Status |
| A/AAAA/CNAME | staging.blog.jeann.net | <ORIGIN ADDRESS> | ON           |
```

or 
```
| Type         | Name                   | Content          | Proxy Status |
| A/AAAA/CNAME | blog.jeann.cf          | <ORIGIN ADDRESS> | ON           |
```

Based on what staging host you're using. The point is to make a staging host and make sure they point to the same origin of `blog.jeann.net` so we can display exact live site at `staging.blog.jeann.net` but still freely test lots of Cloudflare products without impacting real users.

# Make sure you get 200/OK

Completed step 1~2? Then please check if that subdomain responds well to your query. Most of the cases you need additional configurations, because often the origin servers have host check for incoming requests or redirection logic in place. Actually it means your origin doesn't have enough security measures in place if you already see 200/OK without any action in this section... #DevoirAlertForSecurityTeam 

In my case the answer is 404;

{% highlight ruby %}
terminal$ curl -svo /dev/null/ https://staging.blog.jeann.net/ 2>&1 | grep -e "< HTTP" -e "< date"

< HTTP/2 404 
< date: Sun, 31 May 2020 07:58:39 GMT
{% endhighlight %}

This indicates my origin does not have redirection logic for unknown host but it fails to fetch the correct contents to serve. There are origins that does host check and reject the requests coming from unknown hosts too. Well known example is AWS S3. You'll see `403 Forbidden` when trying to connect the assets via unknown hosts. Another example is AWS Cloudfront. If it doesn't regonize a host they'll try to redirect. In any cases, the problem is the origin does not regonize and understand the host and refuses to serve the content.


```
--------------          ---------------------             --------------------
| Request    |          | Cloudflare Proxy   |            | origin server of |
| to staging |          | stg.blog.jeann.net |            | blog.jeann.net   |
--------------          ---------------------             --------------------
       host: stg.blog.jeann.net             host: stg.blog.jeann.net
        -------------------->                -------------------->   
                                                  "origin doesn't understand
                                                    stg.blog.jeann.net."
                                                   returns 404 not found,
                                                           403 forbidden,
                                                           or 301 redirection
```

There are one easy way and one fundamental way to solve this problem and set staging environment.


## Use host header override to trick the origin (ENT ONLY)

This is an easy way. Cloudflare provides a way to tweak http.host value when sending the request back to the origin. You can set it up in Page Rules but the option is enterprise only. In many cases, you'll solve the problem with the page rule.

{% highlight ruby %}
terminal$ curl -svo /dev/null/ https://staging.blog.jeann.net/ 2>&1 | grep -e "< HTTP" -e "< date"

< HTTP/2 200 
< date: Sun, 31 May 2020 08:14:50 GMT
{% endhighlight %}

Why the origin behaviour changed? Because once you set it up, the request will look like below.

```
--------------          ---------------------             --------------------
| Request    |          | Cloudflare Proxy   |            | origin server of |
| to staging |          | stg.blog.jeann.net |            | blog.jeann.net   |
--------------          ---------------------             --------------------
       host: stg.blog.jeann.net                host: blog.jeann.net
        -------------------->                -------------------->   
                                                  "I know this host.
                                                   I can serve the request."
                                                   returns 200 OK
```



Quick and happy if it solves the origin issue. However there're the other ways (e.g. certificate SAN) than host to check if the requests are coming from a foreign host and if the origin still reacts weirdly like keep redirecting you to original host (e.g. Cloudfront) - you need to proceed to the fundamental way.

## Make sure your origin recognize your staging domain name.

This is a fundamental way. You will configure at the origin console to make sure your origin recornizes and acknowledges the staging host you set. You'd need to make sure the origin does serve the contents to your specific foreign host. Look at your cloud console of consult with your web server admin, and make sure you get 200/OK.

# (Optional) Copy Cloudflare configurations to the staging zone

Completed til step 3? Nice! You're all set. Now you have two Cloudflare zones that connect to the same application but they are different because:

while as-is setup does this:

```
-----------------          ---------------------          --------------------
| Request       |          | Cloudflare Proxy  |          | origin server    |
| to production |          | blog.jeann.net    |          | blog.jeann.net   |
-----------------          ---------------------          --------------------
              -------------->
Requesters are              Changes made at this zone
your customers              will affect your customers directly
```

The one we set up will allow you to have below setup, additionally.

```
-----------------          ---------------------          --------------------
| Request       |          | Cloudflare Proxy   |         | origin server    |
| to staging    |          | stg.blog.jeann.net |         | blog.jeann.net   |
-----------------          ---------------------          --------------------
              -------------->
Requesters are                Changes made at this zone
your developers/QA            will not affect your customers
                              and allow you to preview as-is and to-be
```

It is recommended to establish a process on your end to always test first with `staging.blog.jeann.net` before deploying the new product or feature to `blog.jeann.net`.

As you just set up staging environment you may want to copy your dashboard configuration for `blog.jeann.net` onto `staging.blog.jeann.net`. There's no super handy way to do this today(2020-05-31), most of the customers do it manually but if you're familiar with [this well-known configuration management orchestration opensource software](https://developers.cloudflare.com/terraform/), you can manage Cloudflare configuration there. 

# Set up an auth so only your devops can access

Who likes to allow random people to access our company's staging environment? You do not want to open this `staging.blog.jeann.net` up to the public internet. You want to authenticate your devops engineers or QA engineers and make sure only they are able to see staging website not random people outside of the world. There are one intuitive way, and one prettier way to achieve this.

## Use Firewall Rules

This is an intuitive way. You'll specify IP addesses, AS Number, or Cookie, etc to distinguish your developers before they access the application. The rule will look like this.

```
http.host eq "staging.blog.jeann.net" and 
ip.src ne 1.2.3.4 (e.g. your office IP) and 
http.user_agent ne "pre-defined-UA" 
then BLOCK
```

Any Cloudflare customers can use this feature to configure customized security rule. [Read more to learn further.](https://developers.cloudflare.com/firewall/)

## Use Access

This is a prettier way. Without having to deploy anything on your origin, use Cloudflare to force authentication before they access. You will configure the policy first to decide who can see staging website like:

```
jean@yourcompany.com ------------- ALLOW
charlotte@yourcompany.com -------- ALLOW
george@yourcompany.com -----------  DENY
any others at @yourcompany.com --- ALLOW
otherwise ------------------------  DENY
```

And you'll integrate Cloudflare Access with Google or G-suite, Okta, Azure AD, Facebook, whatever you'd like to use for login. So when you access `staging.blog.jeann.net` you'll have to firstly log-in with your devops/QA account.

As of 31 May, it is what is already set in my staging environment: [https://staging.blog.jeann.net/](https://staging.blog.jeann.net/)

Visit [here to learn more](https://developers.cloudflare.com/access/). By the way, you need to have Cloudflare Access in your contract to use this method. If you don't, firewall rules can still help.

# Well done!

Happy playing with experimental features in your Cloudflare staging environment!

There really are lots of features you may turn on to make application cooler and faster!