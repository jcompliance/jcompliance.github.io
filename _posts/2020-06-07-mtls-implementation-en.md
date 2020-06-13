---
layout: post
title:  "Mutual TLS - Only accept clients with valid certificate presented"
ref:  20200607c
date:   2020-06-08 20:58:00 +0800
categories: cloudflare access
tags: cloudflare access security mtls demo
lang: en
---

I'll show how to use Cloudflare Mutual TLS to accept request to certain endpoint(s) only when corresponding client certificate is presented.

Who will be asked to present client certificates are devices and browsers you use. The certificate chain to be used to validate client certificates will be uploaded to Cloudflare mTLS.

Certificate chain is consist of the root cert and the intermediate cert attached in order. I've verified if it works fine with client cert set, by using below openssl query.

{% highlight ruby %}
openssl verify -CAfile certchain.pem clientcert.pem
{% endhighlight %}

You should be able to see the success message as below.

```
clientcert.pem: OK
```

`certchain.pem` is successfully uploaded at Cloudflare mTLS dash, with tied FQDN. I also set a access policy to require valid client certificate when accessing `https://jeann.net/georgetown-penang-malaysia`.

I'll be sending curl.

{% highlight ruby %}
jean$ curl -svo /dev/null/ "https://jeann.net/georgetown-penang-malaysia"

< HTTP/2 403 
< date: Sun, 07 Jun 2020 16:07:06 GMT
< content-type: text/html
{% endhighlight %}

I am blocked by Cloudflare. Now I'll be sending curl with valid client cert.

{% highlight ruby %}
jean$ curl -svo /dev/null/ "https://jeann.net/georgetown-penang-malaysia" --cert clientcert.pem --key clientcert.key

< HTTP/2 200 
< date: Sun, 07 Jun 2020 16:08:31 GMT
< content-type: text/html; charset=UTF-8
{% endhighlight %}

Well authenticated and got 200 OK.

Let's verify using browser too. I'll try accessing the URL without the client certificate.

![](https://jeann.net/wp-content/uploads/2020/06/Screenshot-2020-06-08-at-11.37.31-PM.png)

Blocked by Cloudflare.
So to be able to present a client cert, you should import your cert to your browser first. [Digicert has a well documented how-to which I followed](https://www.digicert.com/kb/managing-client-certificates.htm) and was able to import client cert. If you have difficulties in importing you may want to combine the client cert and the key to PCKS#12 file by using this openssl command - `openssl pkcs12 -export -out jean.p12 -in jean.pem -inkey jean-key.pem`. Anyhow, imported client cert, then try to access the URL which requires a client cert, then you see a browser prompt.

![](https://jeann.net/wp-content/uploads/2020/06/Screenshot-2020-06-08-at-11.46.43-PM.png)
gimme_your_clientcert.jpg

I submit my cert, then able to see the content.

![](https://jeann.net/wp-content/uploads/2020/06/Screenshot-2020-06-08-at-11.47.06-PM.png)

This mutual TLS (2-way-ssl) auth is commonly preferred in banking industry where you would like to require strict client authentication, but not only that, it also supports IoT use cases where you need to auth your things before allowing them connecting to your server. I'd say the benefits of Cloudflare Access Mutual TLS are below;

- Even if you don't have TLS based client auth today but need one, it is easy-to-implement.
- If you do have TLS based client auth today, you can use Cloudflare and enjoy its DDoS protection + Security + Performance benefit together with client auth capability.

# References

- Getting started with mTLS [https://developers.cloudflare.com/access/service-auth/mtls/](https://developers.cloudflare.com/access/service-auth/mtls/)