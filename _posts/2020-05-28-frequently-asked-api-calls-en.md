---
layout: post
title:  "Frequently Asked API Calls (RESTful)"
ref:  20200528a
date:   2020-05-28 09:00:00 +0800
categories: cloudflare analytics
tags: cloudflare api analytics
lang: en
---

I wanted to leave a note on frequently asked API calls (for my personal convenience). They're ready to use, you just have to change below fields to your own requirement:

### Variables
{:.no_toc}

- `<YOUR_EMAIL>` : To specify who's requesting. Use the email address you use to access the dashboard.
- `<API_KEY>` : To authenticate you. Find the key in the dashboard under My Profile - API Tokens - API Keys - Global API Key. Handle this like your password.
- `<ZONE_ID>` : To specify your domain. Find it in the dashboard under Overview - API - Zone ID.
- `<TIME_START>` : Start timestamp formatted as rfc3339 or UNIX. Use UTC time.
- `<TIME_END>` : End timestamp formatted as rfc3339 or UNIX. Use UTC time. When you request logs, you specify "I want logs happened from `<TIME_START>` to `<TIME_END>`" [Details](https://developers.cloudflare.com/logs/logpull-api/requesting-logs/#parameters)

Sections with BETA flags mean they're not part of official offerings yet. Keep in mind that general beta caveats apply - things can be added, changed, paused or withdrawed, also we would appreciate your constructive feedback!

# Table of Contents
{:.no_toc}

* TOC
{:toc}

# Raw Logs (ENT only)

Detailed request logs that were processed at Cloudflare enterprize zone. You may use either Logpush or Logpull to analyse what happened in detailed manner. All API calls under this section is available for enterprise zone only.

## Retrieve Logs

### Logpush

You'll configure the endpoint you want to get the logs saved, then Cloudflare will **push** your logs to that endpoint every 5 minutes. [Get Started](https://developers.cloudflare.com/logs/logpush/)

As of 28/May/2020, below data sets are available.
 - http requests
 - spectrum events

#### Faster Push (BETA)

Up-to-date on: 2020-05-28

Logpush sends you processed logs every 5 minutes. You can make push job even faster which is currently under early access / available via API only. Once enabled, Cloudflare will send log files every 30 seconds instead of 5 minutes. If there was no request made in 30 seconds you won't get the logfile. For your information, the existing Logpush jobs will be updated to this once Beta is completed.

Below are steps to enable faster push. Firstly, run below query to retrieve your existing logpush job ID.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>"     'https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs' | jq
{% endhighlight %}

You'll find the Logpush job id looks like `"id": 14644,`
Save your job id as it will be used at the next query.

Run below query to update current logpush job to faster one (logstream)

{% highlight ruby %}
curl -X PUT "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs/<JOB_ID>" \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" \
     -H "Content-Type: application/json" \
     --data '{"logstream":true}' | jq
{% endhighlight %}

Confirm you got `"logstream": true,` and `"success": true` from the response. Once enabled, you'll receive your logs pushed far more frequently.

If you didn't even configure any logpush but you need this, look at [our developers doc](https://developers.cloudflare.com/logs/logpush/) and [my tutorial](/cloudflare/analytics/2020/05/15/tutorial-cloudflare-logpush.html) to get started. You can use API POST request to create a new Logpush job and already add a necessary option to it.

#### Firewall Events (BETA)

Up-to-date on: 2020-05-28

Although Firewall events are not part of log offerings yet (You can still use GraphQL API to pull events), but as of 28/May/2020, available at Logpush only as early access. You can only enable this data via API.

Run below query to see what data is available.

{% highlight ruby %}
curl -s -X GET https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/datasets/firewall_events/fields \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" | jq .
{% endhighlight %}

If you'd like to try this dataset at your Logpush, please contact your account team (CSM/SE).

#### New Performance Metrics (BETA)

Up-to-date on: 2020-05-28

As part of `http_requests` dataset, the data team at Cloudflare is making new performance metrics available. These fields give much more detail about request timing hence will be helpful to troubleshoot latency issues. This feature is early access, available at Logpush only. Also, you need to enable [Logstream(Look at Faster Logpush)](#faster-push-beta) to use them.

Please contact your account team for more information including fields and definition.

Below are steps to enable these fields. Firstly, run below query to retrieve your existing logpush job ID.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>"     'https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs' | jq
{% endhighlight %}

You'll find the Logpush job id looks like `"id": 14644,` Save your job id as it will be used at the next query. Make sure you're updating `https_requests` not something else.
Also, for the job you want to update, copy what's currently inside at `logpull_options` for e.g.:
{% highlight ruby %}
      "logpull_options": "fields=RayID,ClientIP&timestamps=rfc3339",
{% endhighlight %}

Let's say it's what you currently have, we will add timing metrics into it. They're:
{% highlight ruby %}
fields=EdgeTimeToFirstByteMs,ClientTCPRTTMs,OriginDNSResponseTimeMs,OriginTCPHandshakeDurationMs,OriginTLSHandshakeDurationMs,OriginRequestHeaderSendDurationMs,OriginResponseHeaderReceiveDurationMs,OriginResponseDurationMs,EdgeResponseBodyBytes
{% endhighlight %}

Once you add timing metrics, your updated `logpull_options` will look like below:
{% highlight ruby %}
      "logpull_options": "fields=RayID,ClientIP,EdgeTimeToFirstByteMs,ClientTCPRTTMs,OriginDNSResponseTimeMs,OriginTCPHandshakeDurationMs,OriginTLSHandshakeDurationMs,OriginRequestHeaderSendDurationMs,OriginResponseHeaderReceiveDurationMs,OriginResponseDurationMs,EdgeResponseBodyBytes&timestamps=rfc3339",
{% endhighlight %}

We'll use this to update your existing Logpush. Run the query below.

{% highlight ruby %}
curl -X PUT "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs/<JOB_ID>" \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" \
     -H "Content-Type: application/json" \
     --data '{"logstream": true,"logpull_options":"<UPDATED_LOGPULL_OPTIONS>"}' | jq
{% endhighlight %}

Confirm you got `"success": true` from the response. Once enabled, you'll be able to retrieve performance metrics in your current `http_requests` Logpush job.

If you didn't even configure any logpush but you need this, look at [our developers doc](https://developers.cloudflare.com/logs/logpush/) and [my tutorial](/cloudflare/analytics/2020/05/15/tutorial-cloudflare-logpush.html) to get started. You can use API POST request to create a new Logpush job and already add a necessary fields to it.

#### Add Custom Headers to your Logpush (BETA)

Up-to-date on: 2020-05-28

[On top of these all 50+ fields](https://developers.cloudflare.com/logs/log-fields/#http-requests) Cloudflare Logs/`http_requests` dataset provides, if you wish to add another request headers, response headers, or even cookies to your Logpush, this feature can help you. This is currently under early access.

You need to use Logpush and [also enable Logstream flag](#faster-push-beta) to use this.

If you'd like to try this, firstly, make sure you have a Logpush job and [enable Logstream flag](#faster-push-beta) to your Logpush job. Then contact your account team (CSM/SE), and tell them what fields you want to add and why.

### Logpull

Unlike Logpush, with Logpull you can pull detailed request logs to your choice of servers or laptops by using API queries. You should always [enable log retention](#enable-log-retention) first to go with this option. Use the below query to see if log retention is already on.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/control/retention/flag" | jq .
{% endhighlight %}

Confirm you can see `"flag": true`.

As of 28 May, retention is for last 72 hours (up to 7 days). Do use your storage to regularly pull and save raw logs if you need them for longer period. [Data Retention Period](https://developers.cloudflare.com/logs/logpull-api/understanding-the-basics/#data-retention-period) [Script to pull Cloudflare logs periodically](/cloudflare/analytics/2019/03/20/pulling-cloudflare-logs-with-bash.html)

#### Enable log retention

Source: [https://developers.cloudflare.com/logs/logpull-api/enabling-log-retention/](https://developers.cloudflare.com/logs/logpull-api/enabling-log-retention/)

{% highlight ruby %}
curl -X POST "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/control/retention/flag" -d'{"flag":true}' \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" | jq .
{% endhighlight %}

#### Pull your logs in specific timeframe

You have to [enable log retention](#enable-log-retention) to be able to use this.

This query will pull every available detailed logs for the given timeframe in your current working directory.

{% highlight ruby %}
curl -s \
    -H "X-Auth-Email: <YOUR_EMAIL>" \
    -H "X-Auth-Key: <API_KEY>" \
    "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received?start=<TIME_START>&end=<TIME_END>&fields=$(curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received/fields" | jq '. | to_entries[] | .key' -r | paste -sd "," -)&timestamps=RFC3339" > logpull.json
{% endhighlight %}

With this query, the detailed logs for http requests happened from `<TIME_START>` to `<TIME_END>` will be returned in `logpull.json`.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received/fields
{% endhighlight %}

With this query, you can find what the details are, what fields will be downloaded. Should you need explanation on each field, [please refer to this document.](https://developers.cloudflare.com/logs/log-fields/) Should you need regular pull of your logs than one-off download, [you may want to refer to this bash script.](/cloudflare/analytics/2019/03/20/pulling-cloudflare-logs-with-bash.html)

#### Pull your logs for a specific RayID

You have to [enable log retention](#enable-log-retention) to be able to use this.

This will pull every available detail for the given request, once you put corresponding RayId.

Here, you also need to replace `<RAY_ID>` to the actual RayId (like `59a4189c0813b76f`) you would like to investigate.

Source: [https://developers.cloudflare.com/logs/logpull-api/requesting-logs/#parameters](https://developers.cloudflare.com/logs/logpull-api/requesting-logs/#parameters)

{% highlight ruby %}
curl -X GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/rayids/<RAY_ID>?&fields=$(curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received/fields" | jq '. | to_entries[] | .key' -r | paste -sd "," -)&timestamps=RFC3339" \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" \
     -H "Content-Type: application/json" | jq
{% endhighlight %}

## Analyse Logs

You may have been confused what to do if you got your logs and looking at them, as they don't look friendly readable for human but big flow of data. If you're Cloudflare partner you may need to just go ahead and set up a SIEM to support your enterprise customer. If you're a direct enterprise customer and you think setting up a SIEM for this costs more than the goods, then you may want to just run some simple jq queries to analyse your log files - which is what will be addressed in this section.

Every queries in this section assumes you have used Logpush and Logpull already to get your logs saved in your current working directory. If you don't have logs yet, follow [official Logpush tutorial](https://developers.cloudflare.com/logs/logpush/) or [my Logpush tutorial](/cloudflare/analytics/2020/05/15/tutorial-cloudflare-logpush.html) and download pushed logs in your working directory.

If you'd rather use Logpull, [use this query](#pull-your-logs-in-specific-timeframe) to get your http request logs downloaded.

### Error analysis in specific timeframe

This jq query helps you to find tendency of 4xx, 5xx errors - if the status codes were returned by Cloudflare edge or the origin server.

Firstly, make sure your current working directory has the logs for the timeframe that needs to be analysed.

If your log format follows `.log.gz` run the below query - likely applicable for Logpush.

{% highlight ruby %}
cat *.log.gz | jq -r "(.EdgeResponseStatus|tostring) + \" \" + (.OriginResponseStatus|tostring)" | sort | uniq -c | sort -nr | head -n15
{% endhighlight %}

If your log format follows `.json` run the below query - likely applicable for Logpull.

{% highlight ruby %}
cat *.json | jq -r "(.EdgeResponseStatus|tostring) + \" \" + (.OriginResponseStatus|tostring)" | sort | uniq -c | sort -nr | head -n15
{% endhighlight %}

Once you run the query, you’ll find the output like below.

```
  390 403 0    
  150 200 200
   22 301 301
    8 404 404
    2 403 403
```
The first column represents number of requests, and the next one represents `EdgeResponseStatus` while the third one represents `OriginResponseStatus`. You can see there were 390 requests that were dropped by Cloudflare with 403 Forbidden without touching the origin server. There were only 2 requests that were dropped by the origin server with 403 Forbidden and Cloudflare edge just forwarded 403 to the client. Like this, you can use this query to analyze where exactly the error status came from, if Cloudflare edge or the origin.

### Map each request's clientCountry information to corresponding Cloudflare colo

Cloudflare uses Anycast to route the visitors' request to the neareat available colo, following BGP. This query helps you to analyze if requests from certain country(e.g. Singapore) was serviced in which Cloudflare colo(e.g. Singapore data center, SIN) in certain timeframe.

Firstly, make sure your current working directory has the logs for the timeframe that needs to be analysed.

If your log format follows `.log.gz` run the below query - likely applicable for Logpush. If your logs are formatted in `.json` or `.log`, please modify the query accordingly.

{% highlight ruby %}
cat *.log.gz | jq -r ".ClientCountry + \" \" + .EdgeColoCode" | sort | uniq -c | sort -nr | head -n30
{% endhighlight %}

Once you run the query, you’ll find the output like below.

```
  562 sg SIN
  516 us SEA
  501 xx SEA
  162 xx IAD
  168 us IAD
   83 xx SIN
   81 xx AMS
   78 us EWR
   61 xx EWR
   60 nl AMS
    5 jp NRT
```

The first column represents number of requests, and the next one represents `ClientCountry` while the third one represents `EdgeColoCode`. You can see there were 562 requests were from visitors in Singapore(sg) and were serviced at Cloudflare Singapore colo(SIN), 516 requests were from visitors in the United States(us) and were serviced at Cloudflare Seattle colo(SEA) ... and 5 requests were from visitors in Japan(jp) and were serviced in Cloudflare Tokyo colo(NRT). Like this, you can use this jq to analyze visitors (including attackers) from which country were routed to which Cloudflare colo.

Note: If you're to submit a support ticket based on this query, you need to share the raw log that was used for analysis. If you just refer to something summarized without details, support engineer may get confused.

### PathingStatus information per Status Codes

Firstly, make sure your current working directory has the logs for the timeframe that needs to be analysed.

If your log format follows `.log.gz` run the below query - likely applicable for Logpush. If your logs are formatted in `.json` or `.log`, please modify the query accordingly.

{% highlight ruby %}
cat *.log.gz | jq -r "(.EdgeResponseStatus|tostring) + \" \" + .EdgePathingOp + \" \" + .EdgePathingSrc + \" \" + .EdgePathingStatus" | sort | uniq -c | sort -nr | head -n30
{% endhighlight %}

Once you run the query, you’ll find the output like below.

```
 101 403 ban filterBasedFirewall nr
  74 403 chl user captchaNew
  49 200 wl macro se
  20 304 wl macro nr
  19 301 wl macro mon
  18 503 chl filterBasedFirewall jschlNew
  10 304 wl user nr
  10 200 wl macro nr
   8 200 wl macro unknown
   6 404 wl macro nr
```

The first column represents number of requests, and the next one represents `EdgeResponseStatus` followed by `EdgePathingOp`, `EdgePathingSrc`, `EdgePathingStatus`. [Refer to this documentation to learn further what each value indicates.](https://developers.cloudflare.com/logs/reference/pathing-status/#body-inner)

### Top 20 clientIP, top 20 userAgents in specific timeframe

Firstly, make sure your current working directory has the logs for the timeframe that needs to be analysed.

If your log format follows `.log.gz` run the below query - likely applicable for Logpush. If your logs are formatted in `.json` or `.log`, please modify the query accordingly.

**Top 20 clientIP**

{% highlight ruby %}
cat *.log.gz | jq -r ".ClientIP" | sort | uniq -c | sort -nr | head -n20
{% endhighlight %}

Example output:

```
  129 216.244.66.197
  110 106.75.118.223
  103 106.75.85.117
  ...
```

**Top 20 userAgent**

{% highlight ruby %}
cat *.log.gz | jq -r ".ClientRequestUserAgent" | sort | uniq -c | sort -nr | head -n20
{% endhighlight %}

Example output:

```
  254 Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.112 Safari/537.36
  148 Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)
  130 Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.87 Safari/537.36
  ...
```

As shown, this query will return Top 20 clientIPs and userAgents order by number of requests they sent DESC.

# Web Application Firewall

Cloudflare WAF is a good way for you to stay up-to-date against newly discovered web vulnerabilities as well as already-well-known ones. Even if something is newly found this week, or possibly next week, [Cloudflare WAF team will patch it for your application.](https://developers.cloudflare.com/waf/change-log/) You can't benefit from WAF patch if you didn't turn the WAF on so do make sure your WAF is on.

- As for Cloudflare Managed Ruleset, turn at least two rulesets always on: **Cloudflare Specials** Ruleset and **Cloudflare Miscellaneous** Ruleset. For the others, review if the rulesets descriptions are anything to do with technology stack that's used for your application. If any of them are used, turn them on. [Learn more](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#4vxxAwzbHx0eQ8XfETjxiN)
- As for OWASP ModSecurity Ruleset, it works a bit differently. [Learn more here.](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#sJbboLurEVhipzWYJQnyz)

I'm documenting some frequently asked/answered API queries about WAF API, for my own convenience. If you would like to learn about WAF itself, [here's your go-to-page.](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-)

## WAF Fine Tuning
{:.no_toc}

You just turned Cloudflare WAF on? You guys' end goals will be common - to have proper protection against vulnerabilities - but about how to get started, you may have different opinions.

- You want to follow our recommendation, and also want to maximize detection rate? [Start here.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#minimize-false-negatives)
- You think WAF must not affect any of requests at the moment, you want to review every rule before turning them on? [Start here.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#minimize-false-positives)

## Minimize false negatives

As of 2020-06-06, you have 413 rules in Cloudflare Managed Ruleset and 2491+ rules in OWASP ModSecurity Ruleset, and they are in 'default mode' when you just turned WAF on. Default mode is what Cloudflare recommends with minimum false positives and minimum false negatives, based on our observation at 27M(as of 2020-06-06) applications on the network.

While default mode is our recommendation, the internet is immense, and you will have your application & request flow. So based on your preference and sensitivity you may want to fine-tune WAF rules and it is totally encouraged.

If your thought is close to **'I'd like to start from Cloudflare's recommended mode, but also like to have maximum detection rate'**, this section is yours. Normally applicable when your awareness to security is high.

### Turn on OWASP rules.
{:.no_toc}

[Have a read about how OWASP ruleset works in Cloudflare WAF here)(https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#sJbboLurEVhipzWYJQnyz). In this section we will turn on every-related OWASP rules to your application, to have good detection rate.

Firstly, use below query to retrieve package ID of `OWASP ModSecurity Core Rule Set`.

Source: [https://api.cloudflare.com/#waf-rule-packages-properties](https://api.cloudflare.com/#waf-rule-packages-properties)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages" | jq -r '.result[] | {package: .name, id: .id}'
{% endhighlight %}

Run below query using the `<PACKAGE_ID>` you just retrieved.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>m" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages/<PACKAGE_ID>/groups" | jq '.result[] | {id: .id, name: .name, mode: .mode, count: .rules_count}'
{% endhighlight %}

If some of the OWASP groups in the query result is set as `"mode": "off"`, they need your review. If the rule group looks generic or related to your technology stack used in any way, turn it on, by using one of the below methods.

- UI: Go to Cloudflare dashboard - Firewall - Managed Rules - Package: OWASP ModSecurity Core Rule Set - change modes
- API: Use [this PATCH query](https://api.cloudflare.com/#waf-rule-groups-edit-rule-group) to update the mode to `on`   

You can use below query to verify all necessary rule is set as on.

Source: [https://api.cloudflare.com/#waf-rules-list-rules](https://api.cloudflare.com/#waf-rules-list-rules)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages/<PACKAGE_ID>/rules?per_page=1000&page=1" | jq -cr '.result[] | {id: .id, group: .group.name, mode: .mode} | select(.mode=="off")'
{% endhighlight %}

This will display OWASP rules with mode set as off. Each page can read 1000 rules, but OWASP rules are more than 2000, so you may have to adjust `&page=1` to like, as of 2020-06-06, page 3. You may want to write bash script if you're gonna see this sometimes.

### Adjust sensitivity and action of OWASP rules.
{:.no_toc}

In terms of OWASP sensitivity, [the official doc](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#sJbboLurEVhipzWYJQnyz) recommends to start from `Low`.

> Cloudflare recommends initially setting the WAF Sensitivity to Low and reviewing for false positives before further increasing the Sensitivity.

That is to avoid false positives. However, while customizing some endpoints with false positives during fine-tuning, you will have to gradually increase the sensivity to medium, ultimately high. That's because there are few exploits that are caught in only sensitivity:high.

Same for action. You can start from `action:challenge` or even `simulate`, while customizing some endpoints with false positives during fine-tuning, you will have to set action as `block` as final. `Block` is the action which can actually drop the malicious requests from not only bots, but also humans with bad intention.

You may have unknown bots in your application you have to allow, you may have some unaware requests that WAF does not like, for your whatever business reasons. Do note that `Sensitivity:High` and `Action:Block` gives you the best security but if you set that from Day 1 without any false positive prevention and fine tuning, then you're calling for unpleasant surprises. 

While fine-tuning, look at `ruleId: 981176` events and see if they're legit or not, and proceed with adjustments.

Once fine-tuning is done, the end goal is to have `Sensitivity: High` / `Action: Block` as global set except specific paths that you set.

### Cloudflare WAF 룰의 액션을 최소 Simulate 모드로 올리기
{:.no_toc}

2700만개(2020-06-06 기준) Cloudflare 고객서비스 중, 몇 곳에서 오탐 가능성이 제기되어 Default 동작이 `Disabled`로 되어 있는 룰들이 있습니다. `ruleId 100001: Anomaly:Header:User-Agent - Missing` 이 좋은 예시인데요, user-Agent 필드가 없는 리퀘스트는 보통 정상으로 판단하지 않습니다. 그러나 허술하게 짜여진 커스텀 봇을 다 잡아내기 때문에 이런 봇을 의도적으로 돌리는 몇몇 사이트에서는 오탐으로 판단하실 수도 있습니다. Cloudflare는 2700만개(2020-06-06 기준) 고객서비스를 기준으로 잡으므로 노이즈를 줄이기 위해 이러한 룰들은 로깅 없이 꺼져 있습니다. 고객사 입장에서는 이렇게 꺼져 있는 룰들을 로깅 모드 `Simulate`로 동작으로 바꾸시면, 방어 대상 사이트에 들어오는 리퀘스트의 현재 상태를 알아본 뒤, 이후에 액션 레벨을 높이실지 혹은 낮추실지를 결정하실 수 있습니다.

따라서 여기서는 `Disable`된 룰들을 모두 찾아내고 `Simulate`로 바꾸는 법을 알아볼 것입니다.

먼저, 아래 API를 써서 `Cloudflare Managed Ruleset`의 패키지 ID를 얻어오십시오. 

Source: [https://api.cloudflare.com/#waf-rule-packages-properties](https://api.cloudflare.com/#waf-rule-packages-properties)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages" | jq -r '.result[] | {package: .name, id: .id}'
{% endhighlight %}

아래 쿼리에서 방금 얻어온 `<PACKAGE_ID>` 부분을 붙여넣어 아래 API 콜을 돌리십시오.

Source: [https://api.cloudflare.com/#waf-rules-list-rules](https://api.cloudflare.com/#waf-rules-list-rules)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages/<PACKAGE_ID>/rules?per_page=999&page=1" | jq -cr '.result[] | {id: .id, current_action: .mode, default_action: .default_mode} | select(.default_action=="disable" and .current_action=="default", .current_action=="disable")'
{% endhighlight %}

이 쿼리는 현재 액션이 `disable` 로 설정된 모든 룰과 현재 액션을 표시해 줍니다.

다음은 아래 PATCH 쿼리를 이용하는 Bash Script를 짜셔서 mode를 모두 `simulate`로 바꿔주면 됩니다.

Source: [https://api.cloudflare.com/#waf-rules-edit-rule](https://api.cloudflare.com/#waf-rules-edit-rule)

끝나셨으면 튜닝 준비가 완료되었습니다. 이제부터는 대시보드에서 로깅되는 WAF 룰을 확인해 가시면서 아래 기준으로 튜닝하시면 됩니다.

- 판단에 노이즈가 되는 룰: 공격으로 판단되지 않는데, 지속적으로 대시보드에 출현함 - `Disable`으로 변경 결정
- 더욱 가혹한(?) 처벌이 필요한 룰: 룰을 트리거하는 리퀘스트들이 매우 수상한데, 현재 액션이 `Simulate`나 `Challenge`로 되어 있음 - `Block`으로 변경 결정

## Minimize false positives

Cloudflare WAF를 처음 활성화하시면, 2020-06-06 기준, 413개의 Cloudflare Managed Ruleset과, 2491+개의 OWASP ModSecurity Ruleset이 Default 모드로 동작합니다. Default 모드란 2,700만개의 Cloudflare 고객 서비스를 가장 최저의 오탐율(False Positive)과 미탐율(False Negative)로 보호해 드리기 위한 추천 모드입니다. 

따라서 이 모드를 그대로 활용하시기를 추천하는 한편, 어플리케이션마다의 특성에 맞게, 그리고 고객사의 민감도에 맞게 fine-tuning 하실 수 있도록 Enterprise 고객에 한정하여 지원해 드리고 있습니다.

Fine-tuning의 시작점을, **'실트래픽에 아무런 영향이 없도록, 무조건 시뮬레이션부터 시작하고 싶다'** 로 잡으시는 경우 - 보통 안정성을 가장 중시하시는 서비스는 여기서 시작합니다 - 이 섹션이 도움이 될 것입니다. 이 방식으로 가시는 경우 차단 없이 '탐지 모드' 로 시작합니다. 다만 튜닝에 시간을 오래 끌지 않는 편이 좋습니다. 알려진 익스플로잇도 막지 않는 수준에서 시작하기 때문입니다.

### OWASP WAF 룰을 모두 켜기
{:.no_toc}

먼저, OWASP ModSecurity Core Ruleset에서, 방어 대상 고객서비스와 유관한 모든 구현된 룰들을 켤 것입니다. [이 부분을 참고하세요.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#owasp-waf-%EB%A3%B0%EC%9D%84-%EB%AA%A8%EB%91%90-%EC%BC%9C%EA%B8%B0)

### OWASP 룰의 민감도와 액션을 조정
{:.no_toc}

`Sensitivity: Low` / `Action: Simulate` 로 설정해 주세요.

궁극적으로는 [오탐이 발생하는 특정 endpoint를 튜닝 과정에서 예외 처리하면서](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#waf-룰-세부-조정-ent-only) 민감도를 Medium, 최종적으로는 `High`로 올리는 것이 좋습니다. 민감도 High에서만 발견되는 특정 익스플로잇들이 있기 때문입니다.

액션도 마찬가지입니다. Simulate 모드로 튜닝을 시작하시되, [오탐이 발생하는 특정 endpoint를 튜닝 과정에서 예외 처리하면서](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#waf-룰-세부-조정-ent-only) 최종 Action은 `Block`이 되는 것이 좋습니다. 그래야 OWASP 취약점을 노리는 악성 공격자의 리퀘스트를 실제 차단하실 수 있습니다.

튜닝 과정에서 `ruleId: 981176`으로 표시되는 OWASP 룰과 트리거되는 Path, UA등을 지켜보면서 예외 처리를 진행하세요.

튜닝이 끝나면 예외처리된 특정 Path 외에 전역 룰은 `Sensitivity: High` / `Action: Block` 으로 변경하는 것이 목표입니다.

### Cloudflare WAF 룰의 모든 액션을 Simulate로 변경
{:.no_toc}

Cloudflare WAF는 2700만개(2020-06-06 기준) Cloudflare 고객서비스 대상으로 가장 낮은 미탐율/오탐율을 보이는 추천 액션 설정(Default Mode) 이 있습니다. 각 액션은 아래와 같습니다.

- Disable : 허용
- Simulate : 허용하지만, 이벤트를 로그
- Challenge : CAPTCHA를 통과하면 허용
- Block : 차단

WAF를 소위 **탐지 모드**로 동작시키기 위해 `Challenge`, `Block`, (Optional: `Disable` 포함)으로 동작하는 룰을 모두 `Simulate`로 바꿀 것입니다.

먼저, 아래 API를 써서 `Cloudflare Managed Ruleset`의 패키지 ID를 얻어오십시오. 

Source: [https://api.cloudflare.com/#waf-rule-packages-properties](https://api.cloudflare.com/#waf-rule-packages-properties)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages" | jq -r '.result[] | {package: .name, id: .id}'
{% endhighlight %}

아래 쿼리에서 방금 얻어온 `<PACKAGE_ID>` 부분을 붙여넣어 아래 API 콜을 돌리십시오.

Source: [https://api.cloudflare.com/#waf-rules-list-rules](https://api.cloudflare.com/#waf-rules-list-rules)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages/<PACKAGE_ID>/rules?per_page=999&page=1" | jq -cr '.result[] | {id: .id, current_action: .mode, default_action: .default_mode} | select(.current_action=="block", .current_action=="challenge", (.current_action=="default" and .default_action=="block"), (.current_action=="default" and .default_action=="challenge"))'
{% endhighlight %}

이 쿼리는 현재 액션이 `challenge`나 `block`으로 되어 있는 모든 룰을 표시해 줍니다.

다음은 아래 PATCH 쿼리를 이용하는 Bash Script를 짜셔서 해당 룰의 mode를 모두 `simulate`로 바꿔주면 됩니다.

Source: [https://api.cloudflare.com/#waf-rules-edit-rule](https://api.cloudflare.com/#waf-rules-edit-rule)

끝나셨으면 튜닝 준비가 완료되었습니다. 이제부터는 대시보드에서 로깅되는 WAF 룰을 확인해 가시면서 아래 기준으로 튜닝하시면 됩니다.

- 자주 트리거되는 룰: 탐지되는 개별 룰셋의 정탐 여부를 판단하여 `Block`(이나 `Challenge`)으로 변경 결정
- 판단에 노이즈가 되는 룰: 공격으로 판단되지 않는데, 지속적으로 대시보드에 출현함 - `Disable`으로 변경 결정

## WAF Override (ENT Only)

WAF override allows you to have granular control on WAF rules based on specific URI. As of 2020-06-03, this feature is:

- Enterprise only
- API only

So you should use API. In this example, I will walk you through on how to disable `Cloudflare WAF - OWASP Ruleset` for particular URIs.

### Retrieve current WAF Override rules

Firstly, retrieve the WAF override rules you have in place at the moment.

Source: [https://api.cloudflare.com/#waf-overrides-list-uri-controlled-waf-configurations](https://api.cloudflare.com/#waf-overrides-list-uri-controlled-waf-configurations)

{% highlight ruby %}
curl -X GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/overrides?page=1&per_page=50" \
       -H "X-Auth-Email: <YOUR_EMAIL>" \
       -H "X-Auth-Key: <API_KEY>" \
       -H "Content-Type: application/json" | jq
{% endhighlight %}

If you're just getting started with WAF Override, unlikely you get anything from this.

### Create a new WAF Override rule

Next, create a WAF override rule to bypass OWASP rulesets for your certain endpoint.

Source: [https://api.cloudflare.com/#waf-overrides-create-a-uri-controlled-waf-configuration](https://api.cloudflare.com/#waf-overrides-create-a-uri-controlled-waf-configuration)

- `<RULE_DESCRIPTION>` : Write human-readable rule description. e.g.: Set OWASP to Simulate to API POST endpoints
- `<YOUR_URI>` : List the URIs you want to deploy this change. e.g.: `"api.jeann.net/exampleurl"`. This is an array field so you can put multiple urls. e.g.: `"a.jeann.net/url","b.jeann.net/*","c.jeann.net/rrr*"`. Wildcard is supported.

{% highlight ruby %}
curl -X POST "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/overrides" \
      -H "X-Auth-Email: <YOUR_EMAIL>" \
      -H "X-Auth-Key: <API_KEY>" \
      -H "Content-Type: application/json" \
      --data '{"description":"<RULE_DESCRIPTION>","urls":[<YOUR_URI>],"rules":{"981176":"simulate"}}}}' | jq
{% endhighlight %}

Make sure you see success message from your query. `{"981176":"simulate"}` part will bypass the OWASP ruleset. Cloudflare Managed Ruleset will still protect the URIs. Also, because we're putting this rule to simulate, you'll still be able to see logs from Firewall Events. 

Now, verify if the rule was created successfully, by [using GET query again.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#retrieve-current-waf-override-rules)

### Delete undesired override rules

Source: [https://api.cloudflare.com/#waf-overrides-delete-uri-controlled-waf-configuration](https://api.cloudflare.com/#waf-overrides-delete-uri-controlled-waf-configuration)

No need of WAF Override rule anymore? Then you can delete them using DELETE method. Firstly, [use this query to get the ID of your overrride rule.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#retrieve-current-waf-override-rules)

You should be able to get the ID that looks like below:

`"id": "8594d2aa69e840189ba3328ce3ee5c41"`

Now, run the below query. Replace `<RULE_ID>` to your ID.

{% highlight ruby %}
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/overrides/<RULE_ID>" \
      -H "X-Auth-Email: <YOUR_EMAIL>" \
      -H "X-Auth-Key: <API_KEY>" \
      -H "Content-Type: application/json" | jq
{% endhighlight %}