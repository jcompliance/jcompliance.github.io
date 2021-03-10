---
layout: post
title:  "Practice - Response Headers with Workers"
ref:  20210310a
date:   2021-03-10 07:00:00 +0800
categories: cloudflare workers
tags: cloudflare workers
lang: en
---

{% highlight ruby %}
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {

const url = new URL(request.url)

let response = fetch(request)
response = new Response(response.body, response)

  let cf_flag = request.headers.get("cf-migration") || ''
  if (cf_flag.toLowerCase().startsWith("true")) {
  response.headers.set("cf-workers", 'script_name')
  response.headers.set("cf-url", request.url)
  response.headers.set("cf-cache-ttl", 'defined_ttl')
  response.headers.set("cf-failover-origin", 'origin_host')
}

return response

}{% endhighlight %}