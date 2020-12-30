---
layout: post
title:  "Cloudflare Workers를 이용하여 국가 기반 리디렉션하기"
ref:  c
date:   2020-04-20 22:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare
lang: ko
---

Cloudflare Workers를 사용하면 코드 스니펫을 엣지 데이터 센터에서 돌릴 수 있습니다. 이 예제에서는 리퀘스트 헤더에서 국가 정보를 읽어 블로그의 해당하는 언어 페이지로 리디렉션합니다.

{% highlight ruby %}
addEventListener('fetch', event => {
  event.respondWith(fetchAndApply(event.request))
})

async function fetchAndApply(request) {

  const clientCountry = request.headers.get('CF-IPCountry')
  let colo = request.cf.colo

  if (clientCountry === 'KR') {
    return new Response('', {
        status: 301,
        headers: { 'Location': 'https://blog.jeann.net/kr.html'}
      }
    )}

  else if (clientCountry === 'FR' | colo === 'CDG') { 
       return new Response('', {
        status: 301,
        headers: { 'Location': 'https://blog.jeann.net/fr.html'}
      }
    )}

  else {
      return fetch(request)
  }
}
{% endhighlight %}

이 코드는 국가 정보 (Cloudflare는 2020년 기준 이를 위해 Maxmind를 사용합니다) 와 엣지 데이터센터 (colo) 정보를 읽습니다. 국가 정보가 한국일 경우 리퀘스트는 `/kr.html` 로 리디렉션됩니다. 국가 정보가 프랑스이거나 Cloudflare의 파리 데이터 센터 `CDG`에 히트한 경우 리퀘스트는 `/fr.html`로 리디렉션됩니다. 해당 없을 경우 root path(영문 페이지)로 응답합니다.

이 스니펫은 기본적인 조건부 리디렉션입니다만 방문자가 언어정보를 직접 선택하지 않고도 같은 FQDN 내에서 컨텐츠를 로컬라이즈할 수 있어 널리 사용됩니다. 국가 정보 대신 리퀘스트 헤더의 `accept-language`를 읽어 브라우저가 선호하는 언어셋을 판단하여 리디렉션 하는 것도 고려할 수 있습니다. [이곳에서 코드 스니펫을 참고하세요.](https://blog.jeann.net/cloudflare/workers/2020/12/30/serve-lang-pages-with-workers.html)

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts
