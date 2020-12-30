---
layout: post
title:  "Control Caching with Cloudflare Workers" 
ref:  20201229a
date:   2020-12-30 23:00:00 +0800
categories: cloudflare workers
tags: workers cloudflare caching
lang: en
---

I played with Cloudflare Workers to override basic caching behaviour. The objective was to control caching with Workers only without having to add any page rules.

The code should do the following:

1. If a URL path matches with pre-defined "static extension" regex, regard it static, give TTL (86400s)
2. Even if a URL **does not match the regex**, if the response content type from the cloned response matches with pre-defined "static type", regard it static, give TTL (300s)
3. Shorten cache TTL for specific response status, applicable for static contents only (e.g. 10s if 404)
4. Cache as instucted at 1, 2, 3.
5. Do not cache if not triggered by 1, 2. Even if it matches, never cache one if a method is **not GET**.
6. Add debug headers to confirm TTL and cached status.

The code is as follows:

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

Below is what I learned from writing and testing the code, you may want to read further if you're interested in leveraging it.

Firstly, it is not ideal but I had to fetch the response from the origin first because what I'm trying to do is to tweak the caching behaviour based on the origin response. Then in order to tweak the caching behaviour you end up fetching it again. I wanted to delete the first fetch if ever possible, but I needed it to write the if clause. For the workaround I tried to clone the request so while there're still two subrequests, you have only one original stream. Below:

```
const cRequest = request.clone();
const cResponse = await fetch(cRequest)
const x=cResponse.status
const y=cResponse.headers.get("content-type")
```

Comme ça. I did put 2 fetches but if you do not like it, you can delete the logic based on response status and content-type so you can have only one fetch.

The second part is about Workers product itself. I learned that in the request initiator `cf: cacheEverything: true` works as expected, but not necessarily with `cf: cacheTTl: -1`. It does not guarantee the request will go dynamic. What the hell is it about? Look at the example below:

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

As you see I'm requesting a specific extension [Cloudflare caches by default](https://support.cloudflare.com/hc/en-us/articles/200172516-Understanding-Cloudflare-s-CDN#h_a01982d4-d5b6-4744-bb9b-a71da62c160a) and I'm trying to override it with `cf-cache-ttl: -1` with Workers. Somehow, you still see `cf-cache-status:HIT` in the response header. [According to this community article](https://community.cloudflare.com/t/disable-cache-on-workers-subrequest/122878/3) I am thinking it's Cloudflare Workers' caveat.

> harris / Cloudflare Team / Oct '19<br/>
> There’s currently no way to bypass cache on a fetch() to a static asset from within worker code.

Except these two caveats, you can leverage Workers to control Cloudflare caching instead of adding lots of page rules.

This is my personal template. You can feel free to re-use under your responsibility but please note Cloudflare Solutions Engineers do not:

- Deploy code on customer's production environment
- Commit to any liability regarding the proper behaviour/logic of the code
- Provide code review of customer's scripts

References

- [Clone the request](https://developer.mozilla.org/en-US/docs/Web/API/Request/clone)
- [Disable cache on workers subrequest](https://community.cloudflare.com/t/disable-cache-on-workers-subrequest/122878/3)