---
layout: post
title:  "Try Cloudflare Workers to Accelerate Crawlers"
ref:  craw
date:   2019-09-26 22:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare
lang: en
---

This code snippet is for people who want to increase cache hit rate especially for search engines and crawlers, so they reduce the efforts of hitting the origin server and potentially make it helpful for SEO.

A disclamer for this, this code is pretty experimental. Google has [cloacking policy][google-support] that clarifies if the website is to present different contents or URLs to human users and search engines, it will be considered as cloacking which is violation of webmaster guidelines, and will be depriortised for SEO. This code doesn't aim to display any different contents but it does aim to present cached version of the contents for search engine crawlers. Still there is a risk - understand this before trying. 

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

The code is almost shorter than disclaimer. It's as simple as this - read ASN and UA, determine if it's googlebot, if bingo cache the contents at the edge server and serve from it. If it's NOT googlebot, we are bypassing cache. `cacheTTL: -1` does this. So this code is to deploy to the path you don't intend to cache for human users - if you don't want to tweak anything and put as-is for human users, code can look like below instead.

{% highlight ruby %}
  else {
      return fetch(request)
  }
{% endhighlight %}

This can be helpful for crawler performance because when you look at googlebot's originating location these bots tend to use Chicago (hits Cloudflare's ORD d/c), Seattle (hits Cloudflare's SEA d/c) Taipei (hits Cloudflare's TPE d/c), etc - and [Cloudflare always has closer edge data center][network-map] to googlebot compared to the origin server.

If you have performance issue for crawler but you think caching contents for bots is risky to do for your production - then safer alternative is [Argo Smart Routing][argo-sr]. This will intelligently find faster route than the usual BGP to go to pull dynamic contents from the origin server, so the more far the origin server is from the edge data centers, the more Argo SR can accelerate. Like if your googlebot traffic is originated from somewhere in the US east coast but your origin server is in Korea and you hold lots of dynamic contents - that's exactly when Argo can help.

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts

[google-support]: https://support.google.com/webmasters/answer/66355?hl=en
[network-map]: http://cloudflare.com/network
[argo-sr]: https://www.cloudflare.com/products/argo-smart-routing/