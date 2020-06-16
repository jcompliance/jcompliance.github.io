---
layout: post
title:  "Frequently Asked API Calls (GraphQL)"
ref:  20200529a
date:   2020-05-29 09:00:00 +0800
categories: cloudflare analytics
tags: cloudflare api analytics graphql
lang: en
---

I wanted to leave a note on frequently asked API calls based on GraphQL (for my personal convenience). They're ready to use, you just have to change below fields to your own requirement:

- `<$zone_id>` : To specify your domain. Find it in the dashboard under Overview - API - Zone ID.
- `<$datetime_geq>` : Start timestamp formatted as rfc3339. Use UTC time.
- `<$datetime_leq>` : End timestamp formatted as rfc3339. Use UTC time. When you request logs, you specify "I want logs happened from `<$datetime_geq>` to `<$datetime_leq>`" 

Sections with BETA flags mean they're not part of official offerings yet. Keep in mind that general beta caveats apply - things can be added, changed, paused or withdrawed, also we would appreciate your constructive feedback!

I use GraphiQL as my client program, [same as the official documentation](https://developers.cloudflare.com/analytics/graphql-api/). If you're a very beginner to GraphiQL, [read this article first to get started](https://developers.cloudflare.com/analytics/graphql-api/getting-started/). At GraphiQL, click on `Edit HTTP Headers` and make sure you already put below informations there.

- `X-Auth-Email` : To specify who's requesting. Use the email address you use to access the dashboard.
- `X-Auth-Key` : To authenticate you. Find the key in the dashboard under My Profile - API Tokens - API Keys - Global API Key. Handle this like your password.

Cloudflare GraphQL Endpoint: `https://api.cloudflare.com/client/v4/graphql`

# Table of Contents
{:.no_toc}

* TOC
{:toc}

# Firewall Events

As of 29 May:
- Queryable Duration: Last 31 Days (2,678,400s)
- Sampled in Adaptive Bitrate (Up to 100%)

## Analyse a request in detail

Let's say you have a rayId of a request that looks like blocked by a Cloudflare security service and you'd like to know every detail incl 6W1H of it. Note that this query won't give you a result if it did not trigger Cloudflare Firewall Event. If you don't get a result in this query, [you may want to use Cloudflare Logpull to get detail of a particular rayId.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#pull-your-logs-for-a-specific-rayid)

We need to have RayID to filter by it. Replace `<$rayid>` in this query with the actual RayID.

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zone_id}) {
      firewallEventsAdaptiveGroups (limit:1, filter: $filter, orderBy: [count_DESC])
      {
        dimensions{
          action
          source
          datetime
          clientIP
          clientAsn
          clientCountryName
          edgeColoName
          clientRequestHTTPProtocol
          clientRequestHTTPHost
          clientRequestPath
          clientRequestQuery
          clientRequestScheme
          clientRequestHTTPMethodName
          clientRefererHost
          clientRefererPath
          clientRefererQuery
          clientRefererScheme
          edgeResponseStatus
          clientASNDescription
          userAgent
          kind
          matchIndex
          originResponseStatus
          originatorRayName
          ruleId
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
{
  "zone_id" : "<$zone_id>",
  "filter": {
    "datetime_geq" : "<$datetime_geq>",
    "datetime_leq" : "<$datetime_leq>",
    "rayName" : "<$rayid>"
  }
}
{% endhighlight %}

## Inspect botScore given to a request

As of 2020-05-31, graphql firewall events doesn't provide botScore. Follow the self-documented schema to have up-to-date information. However, [this query will give you botScore and scoring source tied to a given request.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#pull-your-logs-for-a-specific-rayid) Available in Logpush as well. Bring your rayId handy and get useful information including (but not limited to) below:

{% highlight ruby %}
  "BotScore": 2,
  "BotScoreSrc": "Machine Learning",
  "EdgeResponseStatus": 200,
  "OriginResponseStatus": 200,
  "FirewallMatchesActions": [
    "simulate"
  ],
  "FirewallMatchesRuleIDs": [
    "91576b79149c46678682d91daf9ab61f"
  ],
  "FirewallMatchesSources": [
    "firewallRules"
  ],  
{% endhighlight %}

**Applicable only for zones with Bot Management activated.**

[How can botScore help my life? Learn more here](/cloudflare/botmanagement/2020/05/31/are-you-a-bot.html)

## Status codes given per security services and action

As of 2020-05-29, Firewall Events dashboard gives lots of detailed information but you don't see status code information that is given to the blocked request. This query will give you status codes information breakdown when the action given was `drop`, `challenge` or `jschallenge`.

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zone_id}) {
      firewallEventsAdaptiveGroups (limit:20, filter: $filter, orderBy: [count_DESC])
      {
        count
        dimensions{
          action
          source
          kind
          edgeResponseStatus
          originResponseStatus
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
{
  "zone_id" : "<$zone_id>",
  "filter": {
    "datetime_geq" : "<$datetime_geq>",
    "datetime_leq" : "<$datetime_leq>",
    "action_in" : ["drop","challenge","jschallenge"]
  }
}
{% endhighlight %}



## Analyse if Googlebot can reach your website well

It is extremely important to make sure search engine crawlers are able to access fine and crawl contents in your website, without being blocked by Cloudflare security products or anything else. Hence, please follow [this instruction to create a Firewall Rule that allows known good bots](https://developers.cloudflare.com/firewall/known-issues-and-faq/#how-does-firewall-rules-handle-traffic-from-known-bots) first. Once you've done this, we will use a below GraphQL query to analyse if Googlebot can reach your website well, from what locations, etc. 

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zone_id}) {
      firewallEventsAdaptiveGroups (limit:20, filter: $filter, orderBy: [count_DESC])
      {
        count
        dimensions{
          action
          source
          edgeResponseStatus
          originResponseStatus
          edgeColoName
          ruleId
          clientAsn
          userAgent
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
  {
  "zone_id" : "<$zone_id>",
  "filter": {
      "datetime_geq" : "<$datetime_geq>",
      "datetime_leq" : "<$datetime_leq>",
      "clientAsn" : "15169",
      "userAgent_like" : "%Googlebot%"
    }
  }
{% endhighlight %}

Most of the query result should contain `"action": "allow"`. Most of the result should contain `"edgeResponseStatus": 200` as well. For your information, `404` is fine. But if you see Googlebot requests are somehow being blocked, and if you see `403` or `5xx` in HTTP status codes... recheck your security rules and the origin server.

If you don't see anything from the result, that's not good either as it means you aren't getting googlebot requests. (a) check if you created firewall rules correctly, and (b) check if you correctly registered your website at your Google Search Console.


# Cache Analytics (BETA)

As of 29 May:
- Queryable Duration: Last 31 Days (2,678,400s)
- Sampled in Adaptive Bitrate (Up to 1%)

## Analyse a request in detail

Unfortunately, as of 29 May 2020, you can't do this with Cache Analytics because it uses up to 1% adaptive bitrate sampling. Hence, a desired way of using this graphql dataset is to analyse caching tendency. If you'd like to analyse a particular request with rayId in caching context, please use [Cloudflare Logpull.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#pull-your-logs-for-a-specific-rayid)

Still, I'm sharing this query for you to see how the response fields look like in Cache Analytics dataset. This will pick a random request that doesn't have  `CacheStatus` as `"hit"` and returns a detail of it.

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zone_id}) {
      httpRequestsAdaptiveGroups (limit:1, filter: $filter, orderBy: [count_DESC])
      {
        dimensions{
          cacheStatus
          clientAsn
          clientCountryName
          clientDeviceType
          clientIP
          clientRequestHTTPHost
          clientRequestHTTPMethodName
          clientRequestPath
          clientRequestQuery
          clientRequestReferer
          coloCode
          datetime
          edgeResponseContentTypeName
          edgeResponseStatus
          edgeWorkerSubrequestParentRayId
          requestSource
          sampleInterval
          upperTierColoName
          userAgent
          xRequestedWith
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
  {
    "zone_id" : "<$zone_id>",
    "filter": {
      "datetime_geq" : "<$datetime_geq>",
      "datetime_leq" : "<$datetime_leq>",
      "cacheStatus_neq" : "hit"
    }
  }
{% endhighlight %}

## Top 20 items that are uncached

Would you like to increase your cache hit rate to 99% but you are not sure why many of contents remain uncached? Use this query to pull top 20 uncached items in your zone and get explanation on why they were uncached. As of 29 May 2020, this query gives more granularity than Cache Analytics dashboard as it shows request query string.

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zone_id}) {
      httpRequestsAdaptiveGroups (limit:20, filter: $filter, orderBy: [count_DESC])
      {
        count
        dimensions{
          cacheStatus
          clientRequestHTTPHost
          clientRequestPath
          clientRequestQuery
          edgeResponseContentTypeName
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
   {
    "zone_id" : "<$zone_id>",
    "filter": {
      "datetime_geq" : "<$datetime_geq>",
      "datetime_leq" : "<$datetime_leq>",
      "cacheStatus_in" : ["dynamic","none","miss","expired"]
    }
  }
{% endhighlight %}

## Top 20 clientIP, top 20 userAgents in specific timeframe

If you would like to see top 20 clientIP or userAgent etc, in specific timeframe, you can leverage Cache Analytics dataset with few caveats apply.

- As of 2020-06-02, maximum queryable window per query is up to 72 hours(259200s). Hence this cannot serve a requirement such as "I need top 20 client IP info from 1 June to 30 June."
- As of 2020-06-02, Cache Analytics leverages sampled data up to 1%. Keep this in mind when utilizing this data.

Should you need long-term information or you need 100% of data, [please leverage Enterprise Logs instead.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-en.html#top-20-clientip-top-20-useragents-in-specific-timeframe)

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zone_id}) {
      httpRequestsAdaptiveGroups (limit:20, filter: $filter, orderBy: [count_DESC])
      {
        count
        dimensions{
         clientIP
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
   {
    "zone_id" : "<$zone_id>",
    "filter": {
      "datetime_geq" : "<$datetime_geq>",
      "datetime_leq" : "<$datetime_leq>"
    }
  }
{% endhighlight %}

To query userAgent information, please replace `clientIP` to `userAgent` in the query.

# Zone Analytics

Zone Analytics is a traffic overview information which you can find in the Analytics - Traffic app from Cloudflare dashboard. You can query to httpRequest1xGroups (httpRequest1mGroups, httpRequest1hGroups, httpRequest1dGroups) to retrieve information about Zone Analytics.

## Number of requests breakdown by http(s) for a month

If you want to count number of http requests in comparison to https requests made to your domain, you can leverage daily rollup data of the Zone Analytics dataset, which is `httpRequest1dGroups`.

**Query:**
{% highlight ruby %}
query {
  viewer{
    zones(filter: { zoneTag: $zoneid}) {
      httpRequests1dGroups (limit:10, filter: $filter, orderBy: [sum_requests_DESC])
      {
        sum
        {
          requests
          encryptedRequests
        }
      } 
    }
  }
}
{% endhighlight %}

**Query Variables:**
{% highlight ruby %}
{
  "zoneid" : "<$zone_id>",
  "filter": {
    "date_geq" : "2020-05-01",
    "date_leq" : "2020-05-31"
  }
}
{% endhighlight %}