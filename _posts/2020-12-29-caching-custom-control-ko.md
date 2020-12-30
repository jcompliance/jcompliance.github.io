---
layout: post
title:  "Cloudflare Workers로 캐싱 커스텀 제어하기" #"Control Caching with Cloudflare Workers"
ref:  20201229a
date:   2020-12-29 12:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare caching
lang: ko
---

최근 일하고 있는 프로젝트에 필요하여 Cloudflare의 기본 캐싱 동작을 완전히 Override 하는 코드를 만들어 보았다.
Page Rule을 이용하지 않고 Workers 만으로 캐싱을 제어하는 것이 목표이다.

코드가 하는 일은 다음과 같다:

1. URL path에 이용된 확장자가 미리 정의된 "static Extension" 정규표현식에 일치할 경우 정적 콘텐츠로 간주하고 TTL을 부여한다(기본 86400초)
2. 확장자명이 일치하지 **않더라도**, 클론된 Response에서 받아온 content type이 사전 정의된 타입인 경우 정적 콘텐츠로 간주하고 TTL을 부여한다(기본 300초)
3. 1, 2에 의해 정의된 정적 콘텐츠인 경우 특정 Response Status에 대해 TTL을 짧게 감소시킨다. (e.g. 404인 경우 10초)
4. 1, 2, 3에서 정의된 TTL대로 캐시한다.
5. 1, 2에 의해 정의되지 않은 동적 콘텐츠인 경우 캐시하지 않는다. 1, 2에 해당되더라도 메소드가 GET이 아닌 경우에는 캐시하지 않는다.
6. 각 리퀘스트에 대해 캐시 여부와 TTL을 확인할 수 있도록 응답 헤더를 추가한다.

코드는 아래와 같다.

{% highlight ruby %}
addEventListener("fetch", event => {
  event.passThroughOnException()
  event.respondWith(handleRequest(event.request))
})


async function handleRequest(request) {

/* 1: Define static file extensions and default TTL */
const url = new URL(request.url)
const CACHE_CONFIGS = [
   { name: "Static Extension", regex: "^.*\.(bmp|ejs|jpeg|pdf|ps|ttf|class|eot|jpg|pict|svg|webp|css|eps|js|pls|svgz|woff|csv|gif|mid|png|swf|woff2|doc|ico|midi|ppt|tif|xls|docx|jar|otf|pptx|tiff|xlsx|aif|aiff|au|avi|bin|cab|carb|cct|cdf|dcr|dtd|exe|flv|gcf|gff|grv|hdml|hqx|ini|mov|mp3|nc|pct|ppc|pws|swa|txt|vbs|w32|wav|wbmp|wml|wmlc|wmls|wmlsc|xsd|zip|jxr|jp2)", ttl: 86400 }
  ];
const cacheConfigMatched = CACHE_CONFIGS.find( ({regex}) => url.pathname.toLowerCase().match(regex))
let cache = cacheConfigMatched ? cacheConfigMatched : {name:"none", ttl: -1}
const request_method = request.method

console.log("===============================================================================")

let init = { body: request.body,
             method: request.method,
             headers: request.headers,
             redirect: request.redirect,
             fetcher: request.fetcher,
             cf: {  } };

/* cloning the original request to retrieve origin response status & content type and reflect cache settings accordingly. */
const cRequest = request.clone();
const cResponse = await fetch(cRequest)
const x=cResponse.status
console.log("Response Status Code: "+x)
const y= cResponse.headers.get("Content-Type") || cResponse.headers.get("content-type") || ''
console.log("Response Content Type: "+y)
let response=null

/*  2: change cache name and TTL based on response MIME type */
if(cache.name == 'none' && y && y.includes('html'||'json'||'image'||'javascript'||'css'||'plain'||'font'||'audio'||'video')){
  cache.name = 'Switched Static'
  cache.ttl = 300
  console.log("Content Type Match: Switched Static")
}

/*  3: applied to static only: change cache TTL based on response status */
if(cache.name != 'none' && x>=200 && x<=299 && x!=204) {}
  else if(cache.name != 'none' && (x==204||x==305||x==404)) cache.ttl = 10
  else if(cache.name != 'none' && x>=500 && x<=599) cache.ttl = 1
else {}

/*  4 static: define init.cf */
if(cache.name != 'none' && request_method == 'GET') 
{ 
  init.cf = { 
    cacheTtlByStatus: { '200-599': cache.ttl }, 
    cacheEverything: true 
  }
  console.log("Considered static")
}

/* 5 dynamic: define init.cf */
else {
  cache.ttl = -1
  cache.name = 'none'
  init.cf = { cacheTtl: -1 }
  console.log("Considered dynamic")
}

/* get the response based on the defined init.cf */
const subRequest = new Request(request, init);
  console.log(subRequest.cf)
response = await fetch(url, subRequest)
response = new Response(response.body, response)

/* 6: add debug headers */
  response.headers.set("cf-cache-ttl", cache.ttl)
  response.headers.set("cf-cache-rule", cache.name)

return response

}
{% endhighlight %}

아래는 코드 작성과 테스트 과정에서 배운 2가지이니 활용하실 분들께서는 숙지하시기 바란다.

첫째는 Response를 먼저 오리진에서 fetch해 와야 response status와 content type에 맞춰 캐싱 동작을 변화시킬 수 있고 변화시킨 initiator 값에 따라 다시 한번 오리진에서 fetch 해와야 한다는 점이었다. 따라서 fetch를 2번 하게 된다. 가능하면 첫번째 fetch를 삭제했으면 했지만, 첫 fetch를 해오지 않으면 Response status와 content type 정보가 없기에 조건에 따라 다른 TTL을 부여하는 로직을 만들 수 없었다. 그래서 첫 fetch를 그대로 두되 리퀘스트를 clone 하여 subrequest는 2개 발생하지만 원본 스트림은 1개로 유지하는 방식으로 작성하였다. 이 부분이다.

```
const cRequest = request.clone();
const cResponse = await fetch(cRequest)
const x=cResponse.status
const y=cResponse.headers.get("content-type")
```

나는 2개 fetch를 유지하였지만, 1개 fetch로 코드를 깔끔하게 만들고자 하시는 분들은 response status와 content-type 관련 부분을 삭제하시고 fetch를 하나로 유지하시면 된다.

둘째는 Workers 제품 자체에 관계된 부분인데 request initiator 에서 `cf: cacheEverything: true` 구문은 정상 동작하지만, `cf: cacheTTl: -1` 을 주었을 때 꼭 Response가 캐시되지 않으리라는 법은 없다는 점이었다. 이게 뭔 헛소리인고, 하시면 아래 예를 보시라.

{% highlight ruby %}
$ curl -svo /dev/null/ "https://jeann.net/randomly.xlsx"

===============================================================================
worker.js:37 Response Status Code: 404
worker.js:39 Response Content Type: text/html; charset=iso-8859-1
worker.js:69 Considered dynamic
worker.js:74 {cacheTtl: -1}
===============================================================================

404 Not Found
age:0
cache-control:public, max-age=14400
cf-cache-rule:none
cf-cache-status:HIT
cf-cache-ttl:-1
cf-ray:60998ebc192b559e-ORD
cf-request-id:0753d789940000559e42909000000001
content-type:text/html; charset=iso-8859-1
date:Wed, 30 Dec 2020 06:03:25 GMT
{% endhighlight %}

보시다시피, [Cloudflare에서 기본으로 캐시하는 특정 확장자](https://support.cloudflare.com/hc/en-us/articles/200172516-Understanding-Cloudflare-s-CDN#h_a01982d4-d5b6-4744-bb9b-a71da62c160a)의 경우 Workers 스크립트를 통해 동적 컨텐츠로 Override 하려고 해도 여전히 `cf-cache-status:HIT` 정적 자산으로 표기되는 문제가 있다. 이 버그는 기본 캐시 확장자가 아닌 경우에는 발생하지 않는다. [이 커뮤니티 게시글](https://community.cloudflare.com/t/disable-cache-on-workers-subrequest/122878/3)을 봤을 때 Workers 자체의 Caveat이라고 생각되는 부분이다.

> harris / Cloudflare Team / Oct '19<br/>
> There’s currently no way to bypass cache on a fetch() to a static asset from within worker code.

이 2가지 외에는 정상 동작하고 있다. Page Rules에서보다 Workers를 활용하면 개별 룰을 작성하지 않고도 더욱 자유도 높게 캐싱을 활용할 수 있다. 

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts

References

- [리퀘스트 클론하기](https://developer.mozilla.org/en-US/docs/Web/API/Request/clone)
- [Fetch 사용법](https://developer.mozilla.org/ko/docs/Web/API/Fetch_API/Fetch%EC%9D%98_%EC%82%AC%EC%9A%A9%EB%B2%95)
- [Workers 서브리퀘스트에서 캐시 바이패스하기](https://community.cloudflare.com/t/disable-cache-on-workers-subrequest/122878/3)