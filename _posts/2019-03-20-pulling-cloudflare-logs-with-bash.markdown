---
layout: post
title:  "A Bash Script to Pull Cloudflare Raw Logs Periodically"
ref:  20190320
date:   2019-03-20 09:00:00 +0800
categories: cloudflare analytics
tags: logs cloudflare logpull
lang: en
---

Cloudflare raw logs is enterprise only.
Enterprise customers should use [this API][api-logswitch] to enable Cloudflare log retention. That said, once this API is used, CF will start logging and it's not on by default.

This simple bash script will periodically pull every field in Cloudflare raw logs using API. Below fields should be adjusted per your needs.
- `GAP` : here you define when to start pulling logs. In this example we are starting from 5m(300s) ago from current timestamp.
- `INTERVAL` : here you define how often you want to pull logs. In this particular example, the script will work to get logs for 60s(1m).
- `yourzone` : replace this with your zoneId that can be found in the dashboard.
- `youremail` : replace this with your email address for Cloudflare dashboard.
- `yourapikey` : replace this with your glabal api key for the email.

{% highlight ruby %}
#!/bin/bash
  
TIME=`date +%s` #get current time
GAP=240         #set gap for getting log
INTERVAL=60     #set logging interval

ZONE_ID=yourzone 
ADMIN_ID=youremail 
API_KEY=yourapikey

  curl -svo "$TIME.json" -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/logs/received?start=`expr $TIME - $GAP - $INTERVAL`&end=`expr $TIME - $GAP`&timestamps=rfc3339&fields=$(curl -s -H "X-Auth-Email: $AUTH_EMAIL" -H "X-Auth-Key: $API_KEY" "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/logs/received/fields" | jq '. | to_entries[] | .key' -r | paste -sd "," -)" \
  -H "X-Auth-Email: $AUTH_EMAIL" \
  -H "X-Auth-Key: $API_KEY" \
  -H "Content-Type: application/json";

{% endhighlight %}

Next step is to set cronjob to run this bash every `INTERVAL` time then the raw logs will be saved in `$time.json` format. Remember the size limit of each file is 1GB so if you have really a lot of requests coming through your Cloudflare zone, you may need to shorten `INTERVAL` or selectively log specific fields instead of logging everything. This way you can have your logs saved, the next step will be how to utilize and visualize them.

If you are familiar with one of the storages: Google Cloud Storage, AWS S3, Microsoft Azure, or if your SIEM is even Sumo Logic, please don't follow this bash and go straight ahead to Logpush. More intuitive and simple. More read here: [Cloudflare Logpush][doc-logpush]

This guidance is more of a workaround for the ENT users don't have those storages but still need to pull logs to their custom endpoints.

[api-logswitch]: https://developers.cloudflare.com/logs/logpull-api/enabling-log-retention/
[doc-logpush]: https://developers.cloudflare.com/logs/logpush/