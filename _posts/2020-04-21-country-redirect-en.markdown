---
layout: post
title:  "Use Cloudflare Workers for Country Based Redirection"
ref:  c
date:   2020-04-20 22:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare
lang: en
---

If you use Cloudflare Workers, you can run your small code snippet at the edge. In this example, I read country information from request headers and redirected requests to corresponding language page of my blog.

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
      return fetch(request.url, request)
  }
}
{% endhighlight %}

This code reads client country (Cloudflare uses Maxmind for this as of 2020.) and edge data center(colo). If incoming requests are from South Korea, they will be redirected to `/kr.html`. if the requests are from France or they hit Cloudflare's France colo `CDG`, they'll be redirected to `/fr.html`. Other requests are directed to root path.

This snippet is basic form of conditional redirection but is widely used because without having to have visitors to choose the language they can localize the business under the same FQDN. Here, instead of using client country information, reading `accept-language` to determine preferred locale can be considered as well.

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts
