---
layout: post
title:  "Cloudflare WAF Rule Action Change: Disabled -> Simulate"
ref:  20210114a
date:   2021-01-14 14:00:00 +0800
categories: cloudflare api waf
tags: cloudflare api
lang: en
---

{% highlight ruby %}
_CFAPIEMAIL="<YOUR_EMAIL>"
_CFAPIKEY="<API_KEY>"
_CFAPIZONEID="<ZONE_ID>"
 
curl -s -H "X-Auth-Email: $_CFAPIEMAIL" -H "X-Auth-Key: $_CFAPIKEY" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/$_CFAPIZONEID/firewall/waf/packages" | jq -r '.result[]|select(.name == "CloudFlare").id' | \
    while read packageID; do
        for i in $(eval echo "{1..$(curl  -s -H "X-Auth-Email: $_CFAPIEMAIL" -H "X-Auth-Key: $_CFAPIKEY" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/$_CFAPIZONEID/firewall/waf/packages/$packageID/rules?per_page=100" | jq -r '.result_info.total_pages')}"); do
            curl -s -H "X-Auth-Email: $_CFAPIEMAIL" -H "X-Auth-Key: $_CFAPIKEY" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/$_CFAPIZONEID/firewall/waf/packages/$packageID/rules?per_page=100&page=$i" | jq -cr '.result[]' | \
                while read rule; do
                    wafID="$(echo "$rule" | jq -r .id)"
                    defaultMode="$(echo "$rule" | jq -r .default_mode)"
                    if [[ "$defaultMode" == "disable" ]]; then
                        curl -s -H "X-Auth-Email: $_CFAPIEMAIL" -H "X-Auth-Key: $_CFAPIKEY" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/$_CFAPIZONEID/firewall/waf/packages/$packageID/rules/$wafID" -X PATCH --data '{"mode":"simulate"}' | jq
                    fi
                done
            done
        done
{% endhighlight %}