---
layout: post
title:  "URL Redirection with Cloudflare Workers"
ref:  20200530a
date:   2020-05-30 21:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare redirection
lang: en
---

Normally, you use Cloudflare Page Rules to do URL redirection or HTTP -> HTTPS redirection. However, if you have too many number of rules you may encounter the number of rules limit. Below is a simple code snippet to match request protocol and url and handle redirection accordingly.

{% highlight ruby %}
addEventListener('fetch', async event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  let requestURL = new URL(request.url)
  requestProtocol = requestURL.protocol
  requestHost = requestURL.hostname
  requestPath = requestURL.pathname
  requestQuery = requestURL.search

  // replace Always Use HTTPS
  if(requestProtocol === 'http:' && request.url.match(/.*\/\/blog\.jeann\.net\/redirect\/http.*/)){
    return Response.redirect('https://'+requestHost+requestPath, 301)
  }

  // replace Forwarding URL
  else if(request.url.match(/(http|https):\/\/blog\.jeann\.net\/redirect\/url.*/)){
    return Response.redirect('https://jeann.net/wp-content/uploads/2020/04/EWaw0wLUMAUUTq2.jpeg', 302)
  }

  return fetch(request)
}
{% endhighlight %}

As of 2020-05-31, this code is deployed in my `blog.jeann.net/redirect*` route for testing.
Run below query to test HTTP->HTTPS redirection:

{% highlight ruby %}
terminal$ curl -svo /dev/null/ "http://blog.jeann.net/redirect/http?randomquery"
{% endhighlight %}

Expected result:

{% highlight ruby %}
< HTTP/1.1 301 Moved Permanently
< Date: Sun, 31 May 2020 02:41:21 GMT
< Content-Length: 0
< Connection: keep-alive
< Set-Cookie: __cfduid=<REDACTED>; expires=Tue, 30-Jun-20 02:41:21 GMT; path=/; domain=.blog.jeann.net; HttpOnly; SameSite=Lax
< Location: https://blog.jeann.net/redirect/http
< cf-request-id: 030a33fdc40000d9c0bf130200000001
< Set-Cookie: __cf_bm=<REDACTED>; path=/; expires=Sun, 31-May-20 03:11:21 GMT; domain=.blog.jeann.net; HttpOnly; SameSite=None
< Server: cloudflare
< CF-RAY: 59bd55dc6b64d9c0-SIN
{% endhighlight %}

You don't see the headers my origin sends, which means the redirection happens at Cloudflare side.

Run below query to test URL forwarding redirection:

{% highlight ruby %}
terminal$ curl -svo /dev/null/ "https://blog.jeann.net/redirect/urltest?iwantsundae"
{% endhighlight %}

Expected result:

{% highlight ruby %}
< HTTP/1.1 302 Found
< Date: Sun, 31 May 2020 02:45:40 GMT
< Content-Length: 0
< Connection: keep-alive
< Set-Cookie: __cfduid=<REDACTED>; expires=Tue, 30-Jun-20 02:45:40 GMT; path=/; domain=.blog.jeann.net; HttpOnly; SameSite=Lax
< Location: https://jeann.net/wp-content/uploads/2020/04/EWaw0wLUMAUUTq2.jpeg
< cf-request-id: 030a37f1660000dcc2880c0200000001
< Set-Cookie: __cf_bm=<REDACTED>; path=/; expires=Sun, 31-May-20 03:15:40 GMT; domain=.blog.jeann.net; HttpOnly; SameSite=None
< Server: cloudflare
< CF-RAY: 59bd5c2f09d3dcc2-SIN
{% endhighlight %}

Or simply try to visit the link via the browser [https://blog.jeann.net/redirect/url](https://blog.jeann.net/redirect/url) to check redirection works fine.

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts

The code itself is written long time ago. Still working fine, but now Cloudflare official site has a template for [bulk redirects use case](https://developers.cloudflare.com/workers/templates/pages/bulk_redirects/) you may go and refer to that code for your environment as it looks more optimized for bulky urls.
