---
layout: post
title:  "Cloudflare API 쿼리 예제 (GraphQL)"
ref:  20200529a
date:   2020-05-29 09:00:00 +0800
categories: cloudflare analytics
tags: cloudflare api analytics graphql
lang: ko
---

자주 질문하시는 유즈케이스에 대해 바로 사용하실 수 있는 API 쿼리문을 (저의 개인적인 편의를 위해) 모아보려고 합니다. 이 포스트에서는 GraphQL 쿼리만 다룹니다. 샘플 쿼리에 포함되어 있는 아래 필드를 본인 정보로 바꾸시고 복사-붙여넣기 하셔서 사용하시면 바로 사용하실 수 있도록 만들었습니다:

- `<$zone_id>` : 쿼리 대상 도메인의 ID. Cloudflare 대시보드의 Overview - API - Zone ID에서 찾으세요.
- `<$datetime_geq>` : 특정 시간대와 관련된 로그를 뽑으실 때 시작 타임스탬프로 쿼리에 넣는 값입니다. RFC3339 포맷(e.g.: 2020-05-28T00:00:00Z)을 따르며 UTC 시간대를 사용하세요.
- `<$datetime_leq>` : 특정 시간대와 관련된 로그를 뽑으실 때 끝 타임스탬프로 쿼리에 넣는 값입니다. RFC3339 포맷(e.g.: 2020-05-28T01:00:00Z)을 따르며 UTC 시간대를 사용하세요. 

BETA 플래그가 붙은 기능들은 공식 제품/기능의 일부가 아님을 유의하시기 바랍니다. 따라서 일반적인 베타 서비스 주의사항이 적용됩니다 - 기능이 추가되거나, 바뀌거나, 중단되거나 혹은 철회될 수 있습니다. 또한 건설적인 피드백을 주시면 베타 서비스에 큰 도움이 됩니다.

이 예제에서는 쿼리를 보내기 위한 클라이언트 프로그램으로 [공식 기술문서 예제와 동일하게 GraphiQL을 씁니다.](https://developers.cloudflare.com/analytics/graphql-api/) [GraphiQL을 다루는 법에 익숙지 않으시면 여기서 시작하시면 됩니다.](https://developers.cloudflare.com/analytics/graphql-api/getting-started/) GraphiQL 프로그램에서 아래 정보는 `Edit HTTP Headers` 버튼을 누르셔서 미리 입력해 두시는 것이 좋습니다.

- `X-Auth-Email` : 요청 유저의 이메일 주소. Cloudflare 대시보드에 접속하는 이메일 주소를 사용하세요.
- `X-Auth-Key` : 요청 유저 인증 키. Cloudflare 대시보드의 My Profile - API Tokens - API Keys - Global API Key에서 찾으세요. 대시보드 접속 암호와 기본적으로 동일하므로, 키를 안전하게 관리하세요.

Cloudflare GraphQL Endpoint: `https://api.cloudflare.com/client/v4/graphql`

# 목차
{:.no_toc}

* TOC
{:toc}

# Firewall Events

2020년 5월 29일 기준: 
- 최대 쿼리 가능 기간: 최근 31일 (2,678,400초)
- Adaptive Bitrate 샘플링을 이용합니다. (최대 100%)

## 특정 리퀘스트의 디테일한 분석

Firewall Events에 의해 차단된 것으로 보이는 리퀘스트의 RayID를 가지고 있을 때, 어떤 보안 서비스가 왜, 어디서 리퀘스트를 차단한 것인지 분석하는 쿼리입니다. Firewall Event에서 차단된 리퀘스트가 아니면 결과값이 표시되지 않습니다. [이 경우에는 Logpull을 이용해 상세 필드를 분석해 보세요.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#특정-rayid에-대한-로그-풀링하기)

차단된 리퀘스트의 RayID로 필터할 예정이므로 해당 정보를 쿼리의 `<$rayid>` 필드에 입력하시기 바랍니다.

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

## 특정 리퀘스트에 부여된 botScore 분석

2020년 5월 31일 기준, GraphQL Firewall Events 데이터셋은 botScore를 제공하지 않습니다. 추후 바뀔 지 모르니 GraphQL 내 self-documented schema를 확인하세요. 그러나, [이 쿼리를 이용하시면 특정 리퀘스트에 부여된 botScore과 스코어링 소스를 확인할 수 있습니다.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#특정-rayid에-대한-로그-풀링하기) Logpush로도 이용 가능합니다. 분석 대상 리퀘스트의 rayId를 입력하셔서 아래와 같은 (훨씬 많은 필드가 있습니다) 디테일한 정보를 확인하세요:

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

**Bot Management가 활성화된 Zone에만 적용됩니다.**

[Q: 이 필드 어디에 쓰는데요? A: 여기를 읽어 보세요](/cloudflare/botmanagement/2020/05/31/are-you-a-bot-ko.html)

## 각각 보안 제품과 액션에 대한 응답 코드 분석

2020년 5월 29일 기준, Firewall Events 대시보드는 많은 디테일한 정보를 보여주지만 HTTP 응답 코드 정보는 노출되지 않고 있습니다. 하지만 API를 직접 사용하시면 볼 수 있습니다. 이 쿼리를 이용하시면 보안 제품의 액션이  `drop`, `challenge` or `jschallenge`였을 때 나간 HTTP 응답 코드 정보를 정렬하여 보여줍니다.

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

## 구글(네이버)봇 리퀘스트가 사이트에 잘 도달하고 있는지 분석

검색엔진의 리퀘스트가 Cloudflare 보안 제품에 막히지 않고 사이트를 크롤링하는 것은 매우 중요합니다. 그러므로, 먼저 [이 페이지에 있는 지시사항을 따라 known bots을 허용하는 Firewall Rule을 만드십시오.](https://developers.cloudflare.com/firewall/known-issues-and-faq/#how-does-firewall-rules-handle-traffic-from-known-bots) 이후 아래 GraphQL 쿼리를 통해 구글봇이 사이트에 잘 도달하고 있는지, 어느 위치로부터 사이트에 트래픽을 보내고 있는지를 분석할 것입니다.

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

이 쿼리의 결과는 대부분 `"action": "allow"`을 포함하고 있어야 합니다. 또한 대부분의 결과가 `"edgeResponseStatus": 200`을 포함하고 있어야 합니다. 참고로 404는 괜찮습니다. 그러나 구글봇 리퀘스트가 드랍되고 있거나 HTTP 상태 코드가 403, 5xx 등으로 보인다면 보안 룰과 오리진 서버를 확인하십시오.

쿼리 결과가 전혀 돌아오지 않는다면 그것은 그것대로 사이트에 구글봇 크롤링 리퀘스트가 없다는 의미이므로 다소 수상합니다. (a) 구글봇을 허용하는 Firewall Rule이 제대로 만들어져 있는지 확인하십시오. 그리고 (b) 구글 검색 콘솔에서 웹사이트가 제대로 등록되어 있는지 확인하십시오.

덧: 네이버 검색엔진에 대한 분석쿼리가 필요하시면 Query Variables를 아래 내용으로 변경하세요.

**Query Variables:**
{% highlight ruby %}
  {
  "zone_id" : "<$zone_id>",
  "filter": {
      "datetime_geq" : "<$datetime_geq>",
      "datetime_leq" : "<$datetime_leq>",
      "clientAsn" : "23576",
      "userAgent_like" : "%Yeti%"
    }
  }
{% endhighlight %}


# Cache Analytics (BETA)

2020년 5월 29일 기준: 
- 최대 쿼리 가능 기간: 최근 31일 (2,678,400초)
- Adaptive Bitrate 샘플링을 이용합니다. (최대 1%)

## 특정 리퀘스트의 디테일한 분석 

안타깝게도, 2020년 5월 29일 기준 Cache Analytics는 최대 1% 적응형 샘플링 데이터를 이용하기 때문에 이 GraphQL 데이터셋을 잘 이용하시는 방법은 경향성 분석을 하시는 것입니다. 단일 리퀘스트의 디테일한 분석을 하시려면 [Logpull을 이용해 상세 필드를 분석하실 수 있습니다.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#특정-rayid에-대한-로그-풀링하기)

이 쿼리는 Cache Analytics 노드에서 제공하는 필드가 어떤 형태로 리턴되는지 확인하는 용도로 이용하십시오. `CacheStatus`가 `"hit"` 이 아닌 랜덤한 리퀘스트를 하나 골라 해당 리퀘스트에 해당하는 디테일을 리턴하는 쿼리입니다.

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

## 캐시되지 않은 Top 20개 리퀘스트 분석

캐시 적중률을 99%까지 높이고 싶은데 현재 어떤 리퀘스트가 왜 캐시되지 않는지 확신이 서지 않으시나요? 이 쿼리를 이용하면 분석 대상 도메인에 대해 캐시되지 않은 Top 20개 콘텐츠가 리퀘스트 갯수와 함께 표시됩니다. 2020년 5월 29일 기준 API 쿼리를 보내시면 리퀘스트의 쿼리 스트링을 포함하여 대시보드보다 정확한 정보를 보여줍니다.

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

## 특정 시간대에 들어온 리퀘스트의 Top 20 clientIP 혹은 userAgent 분석

특정 시간대에 들어온 리퀘스트의 Top 20 clientIP 정보나, userAgent 정보 등을 보시고 싶다면 Cache Analytics를 이용할 수 있습니다. 몇 가지 주의사항이 적용됩니다.

- 2020-06-02 기준, 최대 쿼리 가능 기간은 72시간(259,200초)입니다. 따라서 "5월 1일부터 31일까지 로그가 필요해요" 등의 요구사항을 해결할 수 없습니다.
- 2020-06-02 기준, 최대 1%의 샘플 데이터를 활용합니다. 감안해서 쿼리하셔야 합니다.

한 달치 정보가 필요하시거나, 100% 데이터를 활용하셔야 한다면, [엔터프라이즈 로그 기능을 사용하셔야 합니다. 이 글을 참고하세요.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#특정-시간대에-들어온-리퀘스트의-top-20-clientip-혹은-useragent-분석)


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

`clientIP`를 `userAgent`로 바꾸시면 userAgent 정보를 질의하실 수 있습니다.

# Zone Analytics

Zone Analytics란 대시보드의 Analytics - Traffic 메뉴에 노출되는 트래픽 개요 정보입니다. httpRequest1xGroups (httpRequest1mGroups, httpRequest1hGroups, httpRequest1dGroups) 데이터셋을 이용하셔서 Zone Analytics 관련 정보를 확인하실 수 있습니다.

## 한 달 동안 들어온 리퀘스트의 https 이용 여부 파악

서비스 도메인에 들어온 전체 리퀘스트 갯수의 http/https 여부를 정리해서 확인하시고 싶으시면, Zone Analytics 데이터의 데일리 롤업 버전인 `httpRequest1dGroups` 데이터셋을 이용하시면 됩니다.

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