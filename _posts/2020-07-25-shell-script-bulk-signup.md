---
layout: post
title:  "Bulk Migrate to Cloudflare DNS"
ref:  20200725a
date:   2020-07-25 18:00:00 +0800
categories: cloudflare api onboarding
tags: cloudflare api
lang: en
---

This shell script will help you to "bulk register" domains to kickstart using Cloudflare DNS, using the API documented here: [https://api.cloudflare.com/#zone-create-zone](https://api.cloudflare.com/#zone-create-zone)

Firstly, make sure you have `zones.txt` in your working directory, with domain you would like to signup with Cloudflare separated by enter key, like;

{% highlight ruby %}
firstdomain.com
seconddomain.co.kr
thirddomain.com.au
fourthdomain.com.ph
fifthdomain.com.sg
{% endhighlight %}

Then run this script.

{% highlight ruby %}
_email="<YOUR_EMAIL>"
_key="<API_KEY>"
_account="<ACCOUNT_ID>"
_type="full"

for i in `cat zones.txt`;do curl -X POST "https://api.cloudflare.com/client/v4/zones" -H "X-Auth-Email: ${_email}" -H "X-Auth-Key:${_key}" -H "Content-Type: application/json" --data "{\"name\":\"$i\",\"account\":{\"id\":\"$_account\"},\"jump_start\":true,\"type\":\"$_type\"}" | jq; done;
{% endhighlight %}

Cloudflare provides [CNAME setup][cname] as well, if you want to bulk signup using CNAME setup you can simply change `_type="full"` part to `_type="partial"`. However, in order to use API signup with CNAME setup Cloudflare team should allow it first, so I _guess_ bulk CNAME signup is for enterprise plan only. 

[cname]: https://support.cloudflare.com/hc/en-us/articles/360020348832-Understanding-a-CNAME-Setup