---
layout: post
title:  "Mutual TLS - 인증서를 제출하는 클라이언트만 접속을 허용하기"
ref:  20200607c
date:   2020-06-07 20:58:00 +0800
categories: cloudflare access
tags: cloudflare access security mtls demo
lang: ko
---

Mutual TLS 를 이용하여 일치하는 Client Certificate가 제출되었을 때만 특정 엔드포인트에 접속이 가능하도록 강제하고자 합니다.

여기서 Client Certificate를 제출하는 대상은 사용하시는 디바이스, 브라우저이며 Client Certificate을 검증할 Certificate Chain이 Cloudflare mTLS에 업로드될 것입니다.

Certificate Chain은 Root Cert와 Intermediate Cert를 순서대로 붙인 것입니다. Client Cert와 잘 동작하는지 여부를, 아래 openssl 코드로 검증했습니다.

{% highlight ruby %}
openssl verify -CAfile certchain.pem clientcert.pem
{% endhighlight %}

검증 메세지가 아래와 같이 보여야 합니다.

```
clientcert.pem: OK
```

`certchain.pem`은 Cloudflare mTLS에 FQDN과 함께 업로드 되었습니다. `https://jeann.net/georgetown-penang-malaysia`에 접속 시도 시 client certificate를 제출하지 않으면 접속이 거부되게끔 정책도 설정 되었습니다.

curl을 보내 봅니다.

{% highlight ruby %}
jean$ curl -svo /dev/null/ "https://jeann.net/georgetown-penang-malaysia"

< HTTP/2 403 
< date: Sun, 07 Jun 2020 16:07:06 GMT
< content-type: text/html
{% endhighlight %}

client cert와 함께 curl을 보내 봅니다.

{% highlight ruby %}
jean$ curl -svo /dev/null/ "https://jeann.net/georgetown-penang-malaysia" --cert clientcert.pem --key clientcert.key

< HTTP/2 200 
< date: Sun, 07 Jun 2020 16:08:31 GMT
< content-type: text/html; charset=UTF-8
{% endhighlight %}

성공적으로 인증되네요.

브라우저로도 확인해 보겠습니다. Client Certificate 없이, URL에 접속해 봅니다.

![](https://jeann.net/wp-content/uploads/2020/06/Screenshot-2020-06-08-at-11.37.31-PM.png)

차단 페이지가 출력됩니다.
브라우저에서 사용하시려면 브라우저에 Client Certificate를 Import해야 합니다. [Digicert에 잘 문서화되어 있어서 그대로 했더니](https://www.digicert.com/kb/managing-client-certificates.htm) Client cert가 추가 되었습니다. 인증서 파일과 키가 분리되어 있으면 잘 불러와지지 않고 openssl 커맨드로 `openssl pkcs12 -export -out jean.p12 -in jean.pem -inkey jean-key.pem` PCKS#12 파일로 합쳐야 잘 불러와집니다. 아무튼 브라우저에서 인증서를 읽을 수 있게 불러온 뒤 Client Cert 인증이 필요한 해당 URL에 다시 접속했더니, 프롬프트가 뜹니다.

![](https://jeann.net/wp-content/uploads/2020/06/Screenshot-2020-06-08-at-11.46.43-PM.png)
인증서_내놓으면_안_잡아먹지.jpg

인증서를 제출해 봅니다. 해당 주소에 멀쩡하게 접속이 됩니다.

![](https://jeann.net/wp-content/uploads/2020/06/Screenshot-2020-06-08-at-11.47.06-PM.png)

이 Mutual TLS (2-way-ssl) 인증 체계는 금융권 등의 강력한 클라이언트 인증(공인인증서가 떠오르신다면, 맞습니다)이 필요한 곳에서 자주 사용하며, IoT 등의 디바이스 인증 유즈케이스도 지원합니다. 

Cloudflare Access Mutual TLS를 이용하시는 것의 이점은:

- TLS 인증서 기반 사용자/디바이스 인증 절차가 없었더라도, 비교적 간단하게 통합 구현이 가능 
- 현재 TLS 인증서 기반 사용자/디바이스 인증 절차가 있는 환경에서도, DDoS나 보안, 성능 등의 Cloudflare 혜택과 클라이언트 인증 기능을 통합하는 것이 가능

정도로 정리할 수 있겠습니다.

2020-06-08 기준, Cloudflare Mutual TLS 는 엔터프라이즈 전용 기능입니다. 직접 엔터프라이즈 고객사에서 테스트해보시길 원하시면 담당 세일즈 대표에게, 엔터프라이즈 고객을 지원하시는 파트너사에서 테스트해보시길 원하시면 담당 채널 파트너 매니저에게 문의하시면 됩니다.

# Reference

- MTLS 설정법 [https://developers.cloudflare.com/access/service-auth/mtls/](https://developers.cloudflare.com/access/service-auth/mtls/)