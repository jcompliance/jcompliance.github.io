---
layout: post
title:  "Serve Language Pages with Cloudflare Workers"
ref:  20201230b
date:   2020-12-30 21:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare redirection
lang: en
---

[Other than country-based language page redirection,](https://stg.blog.jeann.net/cloudflare/workers/2020/04/20/country-redirect-en.html) you may want to read `accept-language` header from the request headers instead. It'll give better information of individuals - if a person connecting from Korea actually prefers English or French, you would want to serve his or her language instead of Korean. This header will reflect the information.

{% highlight ruby %}
addEventListener('fetch', event => {
  event.respondWith(langChange(event.request))
})

async function langChange(request) {

  const clientCountry = request.headers.get('CF-IPCountry')
  const acceptLanguage = request.headers.get('Accept-Language') || request.headers.get('accept-language') || ''
  console.log("Accept-Language: "+acceptLanguage)
  const lang = acceptLanguage.substring(0,2) || ''
  console.log("Language Decision: "+lang)

  if (lang=='en') return fetch('https://blog.jeann.net/en.html')
  else if (lang=='fr') return fetch('https://blog.jeann.net/fr.html')
  else if (lang=='ko') return fetch('https://blog.jeann.net/ko.html')
  else if (lang=='ja') return fetch('https://blog.jeann.net/ja.html')
  else if (lang=='zh') return fetch('https://blog.jeann.net/zh.html')
  else return fetch('https://blog.jeann.net/en.html')

}
{% endhighlight %}

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts

