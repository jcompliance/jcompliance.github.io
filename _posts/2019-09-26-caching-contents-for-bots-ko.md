---
layout: post
title:  "Cloudflare Workers를 이용하여 검색엔진 성능 개선하기"
ref:  craw
date:   2019-09-26 22:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare
lang: ko
---

이 코드 스니펫은 검색엔진 크롤러의 캐시 적중률을 높이고자 하는 사람들을 위한 것입니다. 검색엔진 크롤러의 캐시 적중률을 높이면 오리진 서버에서 직접 콘텐츠를 응답할 필요가 없어 크롤러의 위치가 먼 경우 SEO에 도움이 될 수 있습니다.

다만 이 코드가 실험적이라는 사실에 유의하시기 바랍니다. 구글은 [cloacking policy][google-support] 문서를 통해 일반 유저와 검색엔진 크롤러에 각기 다른 콘텐츠나 URL을 제공하는 것을 금지하고 있습니다. 이 행위는 구글 웹마스터 가이드라인에 위반하는 것으로 간주되어 SEO에 불이익을 받게 됩니다. 이 코드는 일반 유저와 검색엔진 크롤러에 다른 콘텐츠를 제공하는 것을 목적하고 있지 않지만, 일반 유저에게는 오리진 서버에서 직접 콘텐츠를 제공하더라도 검색엔진 크롤러에게는 캐시된 버전을 제공하여 성능을 개선하는 것을 목표로 하고 있습니다. 그러나 구글의 해석에 따라 리스크가 있을 수 있으니 시도하시기 전에 이 부분을 유념하시기 바랍니다.

{% highlight ruby %}
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
 
  let asn = request.cf.asn
  const useragent = request.headers.get('user-agent')
   
 if (asn === '15169' && useragent.includes('googlebot') ){
    console.log(useragent)
    return fetch(request, { cf: { cacheEverything: true },  }   )
  }
 
  else {  return fetch(request, {  cacheTtl: -1  }  )  }
}
{% endhighlight %}

코드가 거의 유의사항만큼 짧습니다. 코드가 하는 일은 단순합니다. 검색엔진 크롤러의 ASN와 user agent를 읽어 리퀘스트가 구글봇인지 판단한 후 일치할 경우 해당 콘텐츠를 캐시하고 엣지 데이터 센터에서 서비스합니다. 구글봇이 아닐 경우 캐시하지 않고 바이패스할 것입니다. `cacheTTL: -1` 부분이 이를 위한 코드입니다. 이 코드는 검색엔진에 중요하지만 캐시하고 싶지 않은 부분에 배포하십시오. 만일 바이패스를 목적으로 하지 않고 일반 유저의 캐시 설정은 그대로 두고 구글봇에만 명시적으로 전체 캐시를 하고 싶은 경우, 마지막 부분의 코드는 아래와 같을 것입니다.

{% highlight ruby %}
  else {
      return fetch(request)
  }
{% endhighlight %}

캐싱을 통해 검색엔진의 크롤링 속도를 높일 수 있습니다. 구글 크롤러의 리퀘스트가 생성되는 위치에는 경향성이 있습니다. Cloudflare의 시카고 데이터 센터(ORD), 시애틀 데이터 센터(SEA), 타이페이 데이터 센터(TPE) 근방 등에서 많이 관찰되며, 이 모든 위치들은 대부분의 경우 고객의 오리진 서버 위치보다 [Cloudflare의 데이터 센터 네트워크][network-map]에서 더 가까울 것이기 때문입니다.

구글 크롤링 성능에 이미 이슈가 있으신 경우, 그러나 크롤러에게 캐시된 버전의 콘텐츠를 제공하는 것이 위험하다고 느끼시는 경우는 [Argo Smart Routing][argo-sr]이 보다 안전한 대안이 될 수 있습니다. 이 기능은 캐시되지 않는 다이나믹 콘텐츠를 오리진에 요청하는 경로에 있어서 일반적인 BGP 경로보다 Cloudflare의 데이터 센터를 거쳐 가는 더 빠른 길이 있다고 판단될 경우 해당 경로를 따라 오리진까지의 리퀘스트를 자동으로 라우팅해 줍니다. 따라서 오리진 서버가 엣지 데이터 센터에서 멀면 멀수록 Argo Smart Routing이 성능을 개선해줄 수 있는 여지도 더 크다고 하겠습니다. 예를 들어 귀하의 사이트의 구글 크롤러가 보통 미국 동부에서 생성된다고 보일 때, 그런데 오리진 서버는 한국에 있을 때, 대부분의 콘텐츠가 캐시 불가능한 다이나믹 콘텐츠일 때 - 이 때 Argo Smart Routing이 크게 도움이 됩니다.

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts

[google-support]: https://support.google.com/webmasters/answer/66355?hl=en
[network-map]: http://cloudflare.com/network
[argo-sr]: https://www.cloudflare.com/products/argo-smart-routing/