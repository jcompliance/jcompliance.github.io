---
layout: post
title:  "Cloudflare API 쿼리 예제 (RESTful)"
ref:  20200528a
date:   2020-05-28 09:00:00 +0800
categories: cloudflare analytics
tags: cloudflare api analytics
lang: ko
---

자주 질문하시는 유즈케이스에 대해 바로 사용하실 수 있는 API 쿼리문을 (개인적인 편의를 위해서) 모아보려고 합니다. 샘플 쿼리에 포함되어 있는 아래 필드를 본인 정보로 바꾸시고 복사-붙여넣기 하셔서 사용하시면 바로 사용하실 수 있도록 만들었습니다:

### 변수
{:.no_toc}

- `<YOUR_EMAIL>` : 요청 유저의 이메일 주소. Cloudflare 대시보드에 접속하는 이메일 주소를 사용하세요.
- `<API_KEY>` : 요청 유저 인증 키. Cloudflare 대시보드의 My Profile - API Tokens - API Keys - Global API Key에서 찾으세요. 대시보드 접속 암호와 기본적으로 동일하므로, 키를 안전하게 관리하세요.
- `<ZONE_ID>` : 쿼리 대상 도메인의 ID. Cloudflare 대시보드의 Overview - API - Zone ID에서 찾으세요.
- `<TIME_START>` : 특정 시간대와 관련된 로그를 뽑으실 때 시작 타임스탬프로 쿼리에 넣는 값입니다. RFC3339 포맷(e.g.: 2020-05-28T00:00:00Z)이나 Unix Timestamp를 따르며 UTC 시간대를 사용하세요.
- `<TIME_END>` : 특정 시간대와 관련된 로그를 뽑으실 때 끝 타임스탬프로 쿼리에 넣는 값입니다. RFC3339 포맷(e.g.: 2020-05-28T01:00:00Z)이나 Unix Timestamp를 따르며 UTC 시간대를 사용하세요. [기술문서](https://developers.cloudflare.com/logs/logpull-api/requesting-logs/#parameters)

BETA 플래그가 붙은 기능들은 공식 제품/기능의 일부가 아님을 유의하시기 바랍니다. 따라서 일반적인 베타 서비스 주의사항이 적용됩니다 - 기능이 추가되거나, 바뀌거나, 중단되거나 혹은 철회될 수 있습니다. 또한 건설적인 피드백을 주시면 베타 서비스에 큰 도움이 됩니다.

# 목차
{:.no_toc}

* TOC
{:toc}

# Cloudflare 로그 (ENT only)

Enterprise Zone에서 처리된 디테일한 리퀘스트 로그를 Logpush나 Logpull 중 한 가지 방법을 이용하여 고객이 상세 분석할 수 있도록 제공해 드리는 기능입니다. 이 섹션의 모든 API 쿼리는 Enterprise Zone에서만 이용 가능합니다.

2020년 5월 28일 기준, 아래 데이터셋이 제공되고 있습니다.
 - http requests
 - spectrum events 

## 로그 받기

### Logpush

Log를 전달받을 endpoint를 설정하시면, Cloudflare에서 매 5분마다 해당 endpoint로 로그를 푸시합니다. [시작하기](https://developers.cloudflare.com/logs/logpush/)

#### 더 빠른 푸시 (BETA)

작성일: 2020-05-28

Logpush를 사용하시면 처리된 로그를 매 5분마다 스토리지로 전송해 드립니다. 그러나, API를 이용해서 5분보다 더 빠르게 로그를 전송받을 수 있습니다. 이 기능은 현재 Early Access이며, 활성화되면 매 5분 대신 매 30초마다 로그파일을 푸시하게 됩니다. 30초 안에 발생한 리퀘스트가 없다면 파일을 푸시하지 않습니다. 참고로, Early Access 기간이 끝나면 모든 Logpush가 매 30초로 업데이트될 예정입니다.

활성화 단계는 아래와 같습니다. 먼저, 아래 쿼리로 현재 설정된 Logpush의 Job ID를 확인하십시오.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>"     'https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs' | jq
{% endhighlight %}

`"id": 14644,` 형태의 ID를 확인하실 수 있습니다.
이 Job ID 는 바로 다음 쿼리에서 사용됩니다.

아래 쿼리로 선택한 Logpush Job을 업데이트하십시오.

{% highlight ruby %}
curl -X PUT "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs/<JOB_ID>" \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" \
     -H "Content-Type: application/json" \
     --data '{"logstream":true}' | jq
{% endhighlight %}

응답값에서 `"logstream": true,`와 `"success": true` 메세지를 확인하세요. 활성화된 후에는 훨씬 빠르게 로그를 받아보실 것입니다.

업데이트할 Logpush Job 자체가 없다면, [Logpush 기술문서](https://developers.cloudflare.com/logs/logpush/)와 [제가 쓴 튜토리얼](/cloudflare/analytics/2020/05/15/tutorial-cloudflare-logpush.html)을 참고하셔서 사용을 시작하세요. API POST Request로 새로운 Logpush Job을 만들면서 바로 필요한 옵션을 추가하실 수도 있습니다.

#### Firewall Events (BETA)

작성일: 2020-05-28

Firewall Events 로그는 2020년 5월 28일 기준 아직 공식 로그 서비스의 일부가 아닙니다만, 현재 베타로 Logpush에 한해 제공되고 있습니다. API로만 켤 수 있습니다.

아래 쿼리로 이 데이터셋에 어떤 필드가 제공되는지 볼 수 있습니다.

{% highlight ruby %}
curl -s -X GET https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/datasets/firewall_events/fields \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" | jq .
{% endhighlight %}

이용해보길 원하시면, 담당 Cloudflare 팀(CSM/SE)에게 연락 주시기 바랍니다.

#### 성능 관련 추가 필드 (BETA)

작성일: 2020-05-28

Cloudflare data team은 성능 관련 추가 필드를 `http_requests` 데이터셋에 추가하고 있습니다. 이 필드들은 리퀘스트 타이밍 정보를 상세하게 제공하여 성능 트러블슈팅에 도움을 줍니다. 현재 베타 서비스이며, Logpush 에서만 이용 가능합니다. `"logstream": true,` 도 설정하셔야 하니 `더 빠른 푸시` 섹션을 확인하세요.

필드 이름과 정의 등의 상세 정보가 필요하신 경우 담당 팀에 연락 주시면 됩니다.

활성화 단계는 아래와 같습니다. 먼저, 아래 쿼리로 현재 설정된 Logpush의 Job ID를 확인하십시오.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>"     'https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs' | jq
{% endhighlight %}

`"id": 14644,` 형태의 ID를 확인하실 수 있습니다. 이 Job ID 는 바로 다음 쿼리에서 사용됩니다.
Dataset 이 `https_requests`이어야 해당 필드를 사용할 수 있음에 유의하십시오.
그리고, 해당 Logpush Job으로부터, 현재 상태의 `logpull_options` 내용을 복사해 두십시오. 예를 들면 아래와 같은 값을 확인할 수 있을 것입니다:

{% highlight ruby %}
      "logpull_options": "fields=RayID,ClientIP&timestamps=rfc3339",
{% endhighlight %}

복사해 둔 현재 옵션에, API 업데이트를 통해 성능 관련 필드를 추가할 것입니다. 아래는 해당 필드들입니다:
{% highlight ruby %}
fields=EdgeTimeToFirstByteMs,ClientTCPRTTMs,OriginDNSResponseTimeMs,OriginTCPHandshakeDurationMs,OriginTLSHandshakeDurationMs,OriginRequestHeaderSendDurationMs,OriginResponseHeaderReceiveDurationMs,OriginResponseDurationMs,EdgeResponseBodyBytes
{% endhighlight %}

현재 옵션에 윗 내용을 복사해 `fields=`의 내용이 성능 관련 필드를 포함하도록 업데이트하십시오. 업데이트된 `logpull_options` 내용은 예를 들면 아래와 같을 것입니다. 해당 내용을 복사해 두십시오.

{% highlight ruby %}
      "logpull_options": "fields=RayID,ClientIP,EdgeTimeToFirstByteMs,ClientTCPRTTMs,OriginDNSResponseTimeMs,OriginTCPHandshakeDurationMs,OriginTLSHandshakeDurationMs,OriginRequestHeaderSendDurationMs,OriginResponseHeaderReceiveDurationMs,OriginResponseDurationMs,EdgeResponseBodyBytes&timestamps=rfc3339",
{% endhighlight %}

이 내용을 추가하여 Logpush Job을 업데이트할 것입니다. 복사해둔 내용을 `<UPDATED_LOGPULL_OPTIONS>`에 붙여넣고, 아래 API 쿼리를 실행하십시오.

{% highlight ruby %}
curl -X PUT "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logpush/jobs/<JOB_ID>" \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" \
     -H "Content-Type: application/json" \
     --data '{"logstream": true,"logpull_options":"<UPDATED_LOGPULL_OPTIONS>"}' | jq
{% endhighlight %}

응답값에서 `"success": true` 메세지를 확인하세요. 활성화된 후에는 현재 `http_requests` Logpush로부터 성능 관련 필드를 받아보실 수 있습니다.

업데이트할 Logpush Job 자체가 없다면, [Logpush 기술문서](https://developers.cloudflare.com/logs/logpush/)와 [제가 쓴 튜토리얼](/cloudflare/analytics/2020/05/15/tutorial-cloudflare-logpush.html)을 참고하셔서 사용을 시작하세요. API POST Request로 새로운 Logpush Job을 만들면서 바로 필요한 필드들을 추가하실 수도 있습니다.

#### 사용자 정의 헤더 로깅 (BETA)

작성일: 2020-05-28

Cloudflare Logs의 `http_requests` 데이터셋에서 [이미 제공해 드리는 50개 이상의 필드 이외에](https://developers.cloudflare.com/logs/log-fields/#http-requests), 다른 리퀘스트 헤더나 응답 헤더, 쿠키 등을 Logpush에 포함하셔야 하는 경우 이 베타 기능을 이용하실 수 있습니다.

이 기능을 이용하시려면 Logpush를 이용하시고 [Logstream 플래그도 설정하셔야 합니다.](#더-빠른-푸시-beta)

따라서 이 기능을 활성화하시려면, 먼저 Logpush job을 만드시고 [Logstream 플래그를 설정하세요.](#더-빠른-푸시-beta) 여기까지 하시고 나서 담당 Cloudflare 팀(CSM/SE)에게 연락 주셔서, 어떤 필드가 어떤 사유로 필요하신지 저희에게 알려주시기 바랍니다.


### Logpull

Enterprise Zone에서 처리된 디테일한 리퀘스트 로그를 API 쿼리를 통해 서버나 컴퓨터에 당겨오는 방식입니다. 이 방식을 이용하시려면 언제나 [로그 저장 활성화](#%EB%A1%9C%EA%B7%B8-%EC%A0%80%EC%9E%A5-%ED%99%9C%EC%84%B1%ED%99%94%ED%95%98%EA%B8%B0)를 사전에 진행하여야 합니다. 아래 쿼리를 통해 이미 로그 저장이 되고 있는지 여부를 확인하십시오.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/control/retention/flag" | jq .
{% endhighlight %}

결과값에서 `"flag": true`를 확인하십시오.

5월 28일 기준, 로그 저장기간은 최근 72시간입니다. (7일까지 저장될 수 있습니다) 더 오랜 기간 저장이 필요하시면 로그를 주기적으로 다운받아서 직접 저장하셔야 합니다. [데이터 저장기간 기술문서](https://developers.cloudflare.com/logs/logpull-api/understanding-the-basics/#data-retention-period) [주기적인 로그 저장을 위한 스크립트](/cloudflare/analytics/2019/03/20/pulling-cloudflare-logs-with-bash.html)

#### 로그 저장 활성화하기

Source: [https://developers.cloudflare.com/logs/logpull-api/enabling-log-retention/](https://developers.cloudflare.com/logs/logpull-api/enabling-log-retention/)

{% highlight ruby %}
curl -X POST "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/control/retention/flag" -d'{"flag":true}' \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" | jq .
{% endhighlight %}

#### 특정 시간대의 로그 풀링하기

[로그 저장 활성화](#%EB%A1%9C%EA%B7%B8-%EC%A0%80%EC%9E%A5-%ED%99%9C%EC%84%B1%ED%99%94%ED%95%98%EA%B8%B0)를 먼저 진행하셔야 하며 활성화하신 이후 발생한 리퀘스트만 다운로드 받아 분석할 수 있습니다.

이 쿼리는 특정 시간대에 발생한 로그의 모든 가능한 디테일을 현재 작업 디렉토리에 다운로드 받습니다.

{% highlight ruby %}
curl -s \
    -H "X-Auth-Email: <YOUR_EMAIL>" \
    -H "X-Auth-Key: <API_KEY>" \
    "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received?start=<TIME_START>&end=<TIME_END>&fields=$(curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received/fields" | jq '. | to_entries[] | .key' -r | paste -sd "," -)&timestamps=RFC3339" > logpull.json
{% endhighlight %}

이 코드를 돌리시면 `<TIME_START>`부터 `<TIME_END>`까지 해당 도메인에 발생한 로그가 `logpull.json` 안에 리턴됩니다.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received/fields
{% endhighlight %}

이 부분만 돌리시면 어떤 필드를 다운받을지 확인할 수 있습니다. 각 필드에 대한 설명은, [기술문서](https://developers.cloudflare.com/logs/log-fields/)를 참고하시기 바랍니다. 일회성 다운로드 대신 지속적으로 로그 저장이 필요한 경우 [Bash Script](/cloudflare/analytics/2019/03/20/pulling-cloudflare-logs-with-bash.html)를 이용하시면 됩니다.

#### 특정 RayID에 대한 로그 풀링하기

[로그 저장 활성화](#%EB%A1%9C%EA%B7%B8-%EC%A0%80%EC%9E%A5-%ED%99%9C%EC%84%B1%ED%99%94%ED%95%98%EA%B8%B0)를 먼저 진행하셔야 하며 활성화 이후 발생한 리퀘스트만 이 쿼리를 이용해 분석할 수 있습니다.

이 쿼리는 대응되는 RayId를 입력하면 해당 리퀘스트에 대해 모든 가능한 디테일을 표시해 줍니다.

아래 쿼리에서 `<RAY_ID>`를 실제 분석하시고자 하는 RayId(`59a4189c0813b76f`과 같은 형태)로 교체하시기 바랍니다.

Source: [https://developers.cloudflare.com/logs/logpull-api/requesting-logs/#parameters](https://developers.cloudflare.com/logs/logpull-api/requesting-logs/#parameters)

{% highlight ruby %}
curl -X GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/rayids/<RAY_ID>?&fields=$(curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/logs/received/fields" | jq '. | to_entries[] | .key' -r | paste -sd "," -)&timestamps=RFC3339" \
     -H "X-Auth-Email: <YOUR_EMAIL>" \
     -H "X-Auth-Key: <API_KEY>" \
     -H "Content-Type: application/json" | jq
{% endhighlight %}

## 로그 분석하기

로그를 받으셨지만 이 수많은 데이터를 받아서 분석해 줄 SIEM이 없으신 경우 어쩔 줄 모르고 계실 수 있습니다. Cloudflare 고객사를 지원하는 파트너사라면, 로그 분석을 지원하기 위한 SIEM의 구축은 종종 필수적입니다만, 직접 고객사인 경우 SIEM을 구축하는 것이 배보다 배꼽이 더 크다면, 다운받은 로그를 가지고 간단한 jq 구문을 활용하여 로그 파일을 분석할 수 있습니다. 바로 이 섹션에서 진행할 내용입니다.

이 섹션의 모든 쿼리는 Logpush나 Logpull을 이용하여 이미 분석 대상 로그가 현재 작업 디렉토리에 `.json` 혹은 `.log.gz` 포맷으로 저장되어 있음을 전제로 합니다. 로그가 저장되어 있지 않다면, Logpush의 경우 [기술문서](https://developers.cloudflare.com/logs/logpush/)와 [제가 쓴 튜토리얼](/cloudflare/analytics/2020/05/15/tutorial-cloudflare-logpush.html)을 참고하셔서 세팅하신 후, 분석을 원하시는 로그를 클라우드 스토리지에서 현재 작업 디렉토리에 다운받도록 하십시오.
Logpull의 경우 [풀링 쿼리](#특정-시간대의-로그-풀링하기)를 이용하여 로그를 다운로드 받으십시오.

### 특정 시간대의 에러량 분석

이 jq 쿼리는 특정 시간대에 4xx, 5xx 등의 에러량이 늘어났을 때 edge에서 리턴한 status code와 origin에서 리턴한 status code를 각각 비교하여 수치를 확인합니다.

먼저 현재 작업 디렉토리에 분석 대상 시간대의 로그를 가지고 있도록 하십시오.

로그 포맷이 `.log.gz` 인 경우 아래 jq를 실행하세요. Logpush인 경우 대부분 이 포맷입니다.

{% highlight ruby %}
cat *.log.gz | jq -r "(.EdgeResponseStatus|tostring) + \" \" + (.OriginResponseStatus|tostring)" | sort | uniq -c | sort -nr | head -n15
{% endhighlight %}

로그 포맷이 `.json` 인 경우 아래 jq를 실행하세요. Logpull인 경우 대부분 이 포맷입니다.

{% highlight ruby %}
cat *.json | jq -r "(.EdgeResponseStatus|tostring) + \" \" + (.OriginResponseStatus|tostring)" | sort | uniq -c | sort -nr | head -n15
{% endhighlight %}

이 쿼리를 실행하면 아래 형태와 같은 output을 얻으실 수 있습니다.

```
  390 403 0    
  150 200 200
   22 301 301
    8 404 404
    2 403 403
```

첫 번째 칼럼은 해당하는 리퀘스트의 갯수이며, 두 번째 칼럼은 `EdgeResponseStatus`, 세 번째 칼럼은 `OriginResponseStatus`입니다. 390개의 리퀘스트는 Origin을 거치지 않고 Cloudflare edge에서 403 Forbidden 응답을 받았으며, 단 2개의 리퀘스트가 Origin에서 403 Forbidden 응답을 직접 받고 Cloudflare edge에서도 Origin 응답에 따라 403을 포워딩한 것을 확인할 수 있습니다. 이런 식으로 이 구문을 통해 각 에러 코드가 edge에서 발생하는지 아니면 오리진에서 리턴된 것인지 판단할 수 있습니다.

### 특정 국가에서 발생한 리퀘스트와 Cloudflare colo 매핑

Cloudflare는 Anycast 방식으로 BGP 룰에 따라 리퀘스트를 네트워크상 가장 가깝고 이용가능한 colo로 라우팅합니다. 이 jq 쿼리는 특정 분석 대상 시간대에 특정 국가(예: 한국)에서 발생한 리퀘스트가 어떤 콜로(예: Cloudflare 한국 데이터센터, ICN)에서 서비스 되었는지 확인하기 위한 구문입니다.

먼저 현재 작업 디렉토리에 분석 대상 시간대의 로그를 가지고 있도록 하십시오.

로그 포맷이 `.log.gz` 인 경우 아래 jq를 실행하세요. Logpush인 경우 대부분 아래 포맷입니다. `.json`이나 `.log` 포맷을 사용하시는 경우 쿼리를 변경하시기 바랍니다.

{% highlight ruby %}
cat *.log.gz | jq -r ".ClientCountry + \" \" + .EdgeColoCode" | sort | uniq -c | sort -nr | head -n30
{% endhighlight %}

이 쿼리를 실행하면 아래 형태와 같은 output을 얻으실 수 있습니다.

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

첫 번째 칼럼은 해당하는 리퀘스트의 갯수이며, 두 번째 칼럼은 `ClientCountry`, 세 번째 칼럼은 `EdgeColoCode`입니다. 562개의 리퀘스트는 싱가폴(sg)에서 발생하여 싱가폴 colo(SIN)에서 서비스 되었고, 516개의 리퀘스트는 미국(us)에서 발생하여 미국 시애틀 colo(SEA)에서 서비스 되었으며 ... 5개 리퀘스트는 일본(jp)에서 발생하여 도쿄 colo(NRT)에서 서비스 되었다고 이해할 수 있습니다. 이런 식으로 이 구문을 통해 각 국가의 방문자 (+공격자 포함) 리퀘스트가 어떤 Cloudflare colo에 도착하였는지 판단할 수 있습니다.

주의사항: 이 쿼리를 기반으로 지원 티켓을 제출하려는 경우 분석에 사용된 로그를 함께 공유하십시오. 디테일이 없는 요약된 결과만 가지고는 지원 엔지니어가 혼란에 빠질 수 있습니다.

### 각 Status code별로 리턴된 PathingStatus 정보

먼저 현재 작업 디렉토리에 분석 대상 시간대의 로그를 가지고 있도록 하십시오.

로그 파일들이 저장된 디렉토리에서 아래 jq를 실행하세요. Logpush인 경우 대부분 아래 포맷입니다. `.json`이나 `.log` 포맷을 사용하시는 경우 쿼리를 변경하시기 바랍니다.

{% highlight ruby %}
cat *.log.gz | jq -r "(.EdgeResponseStatus|tostring) + \" \" + .EdgePathingOp + \" \" + .EdgePathingSrc + \" \" + .EdgePathingStatus" | sort | uniq -c | sort -nr | head -n30
{% endhighlight %}

이 쿼리를 실행하면 아래 형태와 같은 output을 얻으실 수 있습니다.

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

첫 번째 칼럼은 해당하는 리퀘스트의 갯수이며, 두 번째 칼럼은 edge에서 리턴된 스테이터스 코드입니다. 그 뒤부터는 각각 `EdgePathingOp`, `EdgePathingSrc`, `EdgePathingStatus` 정보를 표시해 줍니다. [이 문서를 참고하셔서 의미를 해석하세요.](https://developers.cloudflare.com/logs/reference/pathing-status/#body-inner)

### 특정 시간대에 들어온 리퀘스트의 Top 20 clientIP 혹은 userAgent 분석

먼저 현재 작업 디렉토리에 분석 대상 시간대의 로그를 가지고 있도록 하십시오.

로그 파일들이 저장된 디렉토리에서 아래 jq를 실행하세요. Logpush인 경우 대부분 아래 포맷입니다. `.json`이나 `.log` 포맷을 사용하시는 경우 쿼리를 변경하시기 바랍니다.

**Top 20 clientIP**

{% highlight ruby %}
cat *.log.gz | jq -r ".ClientIP" | sort | uniq -c | sort -nr | head -n20
{% endhighlight %}

출력 예:

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

아웃풋 예:

```
  254 Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.112 Safari/537.36
  148 Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)
  130 Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.87 Safari/537.36
  ...
```

이와 같이 이 쿼리는 가장 많은 리퀘스트를 보낸 순서대로 Top 20 clientIP와 userAgent 값을 리턴할 것입니다.

# Web Application Firewall

Cloudflare WAF는 여러 알려진 웹 취약점 뿐만이 아니라 바로 이번 주, 다음 주에도 새롭게 발견될 수 있는 취약점 익스플로잇에 대한 방비를 갖출 수 있는 좋은 방법입니다. [WAF를 켜두기만 해도 Cloudflare의 최신 패치의 혜택을 받을 수 있습니다](https://developers.cloudflare.com/waf/change-log/). 고객이 켜지 않으면 혜택을 받을 수 없으므로, WAF를 늘 켜두도록 하십시오.

- Cloudflare Managed Ruleset의 경우, **Cloudflare Specials** Ruleset과 **Cloudflare Miscellaneous** Ruleset을 늘 켠 상태로 두십시오. 나머지 Ruleset의 경우 귀사의 서비스에 사용된 기술 스택과 관계가 있는지 검토하셔서 관계가 있으면 꼭 켜도록 하십시오. [공식 기술 문서 읽어보기](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#4vxxAwzbHx0eQ8XfETjxiN)
- OWASP ModSecurity Ruleset의 경우, [룰 동작 방식이 조금 다릅니다. 공식 기술 문서 읽어보기](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#sJbboLurEVhipzWYJQnyz).

여기서는 WAF의 API 활용 관련 자주 질문하시는 쿼리를 (개인적인 편의를 위해) 문서화 해보려고 합니다. WAF 자체에 대한 자세한 내용은 [기술문서에서 확인하세요](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-).

## WAF Fine Tuning
{:.no_toc}

Cloudflare WAF를 방금 켜셨나요? WAF를 적용하고자 하는 고객사는 모두 웹취약점을 방어해야 한다는 인식이 있을 것이지만, 어떻게 적용을 시작할지에 대해서는 의견이 다를 수 있을 것입니다.

- 추천설정 + 최대 탐지율로 시작하셔서 노이즈를 줄여가는 방식을 원하시면, [여기서부터 시작하세요.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#%EB%AF%B8%ED%83%90false-negative-%EC%A4%84%EC%9D%B4%EA%B8%B0)
- 현재의 트래픽에 아무 영향을 주지 않으면서, 하나씩 탐지 룰을 켜가는 방식을 원하시면, [여기서부터 시작하세요.](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#%EC%98%A4%ED%83%90false-positive-%EC%A4%84%EC%9D%B4%EA%B8%B0)

## 미탐(False Negative) 줄이기

Cloudflare WAF를 처음 활성화하시면, 2020-06-06 기준, 413개의 Cloudflare Managed Ruleset과, 2491+개의 OWASP ModSecurity Ruleset이 Default 모드로 동작합니다. Default 모드란 2,700만개의 Cloudflare 고객 서비스를 가장 최저의 오탐율(False Positive)과 미탐율(False Negative)로 보호해 드리기 위한 추천 모드입니다. 

따라서 이 모드를 그대로 활용하시기를 추천하는 한편, 어플리케이션마다의 특성에 맞게, 그리고 고객사의 민감도에 맞게 fine-tuning 하실 수 있도록 Enterprise 고객에 한정하여 지원해 드리고 있습니다.

Fine-tuning의 시작점을, **'추천설정으로 시작하되, 추천설정에서 잡히지 않는 공격도 다 캐치하고 싶다'** 로 잡으시는 경우 - 보통 보안에 대한 awareness가 높으신 경우 여기서 시작합니다 - 이 섹션이 도움이 될 것입니다.

### OWASP WAF 룰을 모두 켜기
{:.no_toc}

OWASP WAF 룰셋의 특징을 [이 기술 문서에서 읽어 보시기 바랍니다](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#sJbboLurEVhipzWYJQnyz). 이 섹션에서는 OWASP가 catch-all을 잘 하게 하기 위해, 고객서비스와 유관한 모든 구현된 룰들을 켤 것입니다.

먼저, 아래 API를 써서 `OWASP ModSecurity Core Rule Set`의 패키지 ID를 얻어오십시오. 

Source: [https://api.cloudflare.com/#waf-rule-packages-properties](https://api.cloudflare.com/#waf-rule-packages-properties)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages" | jq -r '.result[] | {package: .name, id: .id}'
{% endhighlight %}

아래 쿼리에서 방금 얻어온 `<PACKAGE_ID>` 부분을 붙여넣어 아래 API 콜을 돌리십시오.

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>m" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages/<PACKAGE_ID>/groups" | jq '.result[] | {id: .id, name: .name, mode: .mode, count: .rules_count}'
{% endhighlight %}

이 쿼리의 결과 중 `"mode": "off"` 를 보시면 기본적으로 꺼져 있는 OWASP 룰셋을 확인할 수 있습니다. 꺼져 있는 룰셋이 귀사의 서비스에 사용된 기술 스택과 관계가 없다면 그대로 두십시오. 일반적인 룰셋에 해당되거나 귀사의 서비스에 사용된 기술 스택과 관계가 있다면 아래 방법 중 한 가지를 이용하여 켜도록 하십시오.

- UI: Cloudflare 대시보드 - Firewall - Managed Rules - Package: OWASP ModSecurity Core Rule Set에서 모드 변경
- API: [이 PATCH 쿼리](https://api.cloudflare.com/#waf-rule-groups-edit-rule-group)를 이용하셔서 mode를 on으로 업데이트 하시면 됩니다.

필요한 모든 룰이 잘 켜졌는지 확인하기 위해 아래 쿼리를 이용할 수 있습니다.

Source: [https://api.cloudflare.com/#waf-rules-list-rules](https://api.cloudflare.com/#waf-rules-list-rules)

{% highlight ruby %}
curl -s -H "X-Auth-Email: <YOUR_EMAIL>" -H "X-Auth-Key: <API_KEY>" -H "Content-Type: application/json" "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/packages/<PACKAGE_ID>/rules?per_page=1000&page=1" | jq -cr '.result[] | {id: .id, group: .group.name, mode: .mode} | select(.mode=="off")'
{% endhighlight %}

mode가 off인 룰들을 표시해 주는 쿼리입니다. 한 페이지당 1000개까지 표시할 수 있는데 OWASP 룰은 수가 많기 때문에, 꺼져있는 전체 룰을 표시하려면 `&page=1` 부분을 변경하셔서 2020-06-06 기준 3페이지까지는 보셔야 합니다. 귀찮으시면, Bash Script를 써서 돌리면 한 번 콜로 효율화 할 수가 있겠죠. 

### OWASP 룰의 민감도와 액션을 조정
{:.no_toc}

[공식 기술문서](https://support.cloudflare.com/hc/en-us/articles/200172016-Understanding-the-Cloudflare-Web-Application-Firewall-WAF-#sJbboLurEVhipzWYJQnyz)에서는 OWASP 룰의 민감도를 Low로 시작하기를 권장하고 있습니다.

> Cloudflare recommends initially setting the WAF Sensitivity to Low and reviewing for false positives before further increasing the Sensitivity.

오탐을 줄이기 위해서입니다. 그러나, 궁극적으로는 [오탐이 발생하는 특정 endpoint를 튜닝 과정에서 예외 처리하면서](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#waf-룰-세부-조정-ent-only) 민감도를 Medium, 최종적으로는 `High`로 올리는 것이 좋습니다. 민감도 High에서만 발견되는 특정 익스플로잇들이 있기 때문입니다.

액션도 마찬가지입니다. `Challenge` 혹은 `Simulate` 모드로 튜닝을 시작하시되, [오탐이 발생하는 특정 endpoint를 튜닝 과정에서 예외 처리하면서](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#waf-룰-세부-조정-ent-only) 최종 Action은 `Block`이 되는 것이 좋습니다. 그래야 OWASP 취약점을 노리는 악성 공격자의 리퀘스트를 실제 차단하실 수 있습니다.

귀사의 어플리케이션에 예상치 못한 봇들이 돌고 있거나, 비즈니스적 이유로 WAF가 좋아하지 않을 만한 리퀘스트가 돌고 있을 수 있습니다. `Sensitivity:High`와 `Action:Block`은 최상의 보안을 제공합니다만 오탐 방지와 fine tuning 과정 없이 Day 1부터 적용했다가는 unpleasant surprises를 겪으실 수 있다는 점 유의하세요.

튜닝 과정에서 `ruleId: 981176`으로 표시되는 OWASP 룰과 트리거되는 Path, UA등을 지켜보면서 예외 처리를 진행하세요.

튜닝이 끝나면 예외처리된 특정 Path 외에 전역 룰은 `Sensitivity: High` / `Action: Block` 으로 변경하는 것이 목표입니다.

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

## 오탐(False Positive) 줄이기

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

## WAF 룰 세부 조정 (ENT Only)

WAF Override를 이용하시면 개별 URI에 대한 WAF 룰 동작을 세부 조정할 수 있습니다. 2020-06-03 기준, 이 기능은:

- Enterprise 전용
- API 전용

입니다. 따라서, API로만 기능 이용이 가능합니다. 이 예시에서는 특정 URI에 대해서만 `Cloudflare WAF - OWASP Ruleset`를 끄는 법을 설명할 것입니다.

### 현재 설정된 WAF Override 룰 확인

먼저, 현재 설정된 WAF Override 룰을 확인할 것입니다.

Source: [https://api.cloudflare.com/#waf-overrides-list-uri-controlled-waf-configurations](https://api.cloudflare.com/#waf-overrides-list-uri-controlled-waf-configurations)

{% highlight ruby %}
curl -X GET "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/overrides?page=1&per_page=50" \
       -H "X-Auth-Email: <YOUR_EMAIL>" \
       -H "X-Auth-Key: <API_KEY>" \
       -H "Content-Type: application/json" | jq
{% endhighlight %}

해당 서비스에 WAF Override 적용한 적이 아직 없다면, 아무 결과도 리턴되지 않을 것입니다.

### 새로운 WAF Override 룰 설정

다음으로, 특정 엔드포인트에 한해 OWASP 룰셋을 바이패스하는 WAF Override 룰을 만듭니다.

Source: [https://api.cloudflare.com/#waf-overrides-create-a-uri-controlled-waf-configuration](https://api.cloudflare.com/#waf-overrides-create-a-uri-controlled-waf-configuration)

- `<RULE_DESCRIPTION>` : 자유롭게 (영어로) 룰 설명을 작성하십시오. e.g.: Set OWASP to Simulate to API POST endpoints
- `<YOUR_URI>` : WAF Override를 적용할 URI를 나열하십시오. e.g.: `"api.jeann.net/exampleurl"`. 어레이 필드이므로 여러 URI를 작성할 수 있습니다. e.g.: `"a.jeann.net/url","b.jeann.net/*","c.jeann.net/rrr*"`. 와일드카드가 지원됩니다.

{% highlight ruby %}
curl -X POST "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/overrides" \
      -H "X-Auth-Email: <YOUR_EMAIL>" \
      -H "X-Auth-Key: <API_KEY>" \
      -H "Content-Type: application/json" \
      --data '{"description":"<RULE_DESCRIPTION>","urls":[<YOUR_URI>],"rules":{"981176":"simulate"}}}}' | jq
{% endhighlight %}

응답값에서 success 메세지를 확인하십시오. `{"981176":"simulate"}` 부분이 OWASP 룰셋을 바이패스할 것입니다. OWASP 룰셋을 끄더라도 Cloudflare Managed Ruleset은 해당 URI들에 계속해서 적용됩니다. 그리고 액션을 simulate로 바꾸었기 때문에 여전히 Firewall Events에서 이벤트를 확인할 수 있습니다.

다음으로, [이 GET 쿼리를 다시 이용](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#%ED%98%84%EC%9E%AC-%EC%84%A4%EC%A0%95%EB%90%9C-waf-override-%EB%A3%B0-%ED%99%95%EC%9D%B8)하셔서 룰이 정상적으로 만들어졌는지 확인하십시오.

### 필요 없는 WAF Override 룰 삭제

Source: [https://api.cloudflare.com/#waf-overrides-delete-uri-controlled-waf-configuration](https://api.cloudflare.com/#waf-overrides-delete-uri-controlled-waf-configuration)

WAF Override 룰이 더 이상 필요하지 않다면, DELETE 메소드를 써서 지울 수 있습니다. 먼저, [GET 쿼리를 이용하여 삭제 대상 룰의 ID를 확인](/cloudflare/analytics/2020/05/28/frequently-asked-api-calls-ko.html#%ED%98%84%EC%9E%AC-%EC%84%A4%EC%A0%95%EB%90%9C-waf-override-%EB%A3%B0-%ED%99%95%EC%9D%B8)하십시오.

아래와 같은 형식의 ID를 확인할 수 있습니다.

`"id": "8594d2aa69e840189ba3328ce3ee5c41"`

`<RULE_ID>`를 필요한 ID로 바꿔넣고, 아래 쿼리를 돌리세요.

{% highlight ruby %}
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/firewall/waf/overrides/<RULE_ID>" \
      -H "X-Auth-Email: <YOUR_EMAIL>" \
      -H "X-Auth-Key: <API_KEY>" \
      -H "Content-Type: application/json" | jq
{% endhighlight %}