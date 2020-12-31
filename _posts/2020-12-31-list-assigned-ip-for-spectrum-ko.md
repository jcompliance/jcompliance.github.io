---
layout: post
title:  "고객사의 전체 Spectrum App의 IP 조사하기" 
ref:  20201231a
date:   2020-12-31 18:00:00 +0800
categories: cloudflare api spectrum
tags: spectrum cloudflare api
lang: ko
---

이 스크립트는 Cloudflare가 Spectrum 호스트별로 동적 할당하는 IP주소를 모니터링하여 Network Analytics상에서 보이는 Flood 공격이 정확히 어떤 호스트로 향하고 있는지 판단하기 위해서 활용할 수 있습니다. 스크립트는:

1. Cloudflare account에 등록된 모든 도메인(zone)을 불러오고

2. 각각 도메인(zone)에 등록된 Spectrum host를 모두 불러오고

3. 그 host에 각각 할당된 IP를 쿼리하여 리턴합니다.

Cloudflare API를 활용하므로 이용하시는 유저의 Cloudflare Account ID, email, global API key를 입력하셔야 합니다.

{% highlight ruby %}
#!/bin/bash

i=0; 
cnt=0;

  echo "Account ID?"
  read accountID
 
  echo "Your email?"
  read email

  echo "Your API Key?"
  read apikey

printf "Now reading the whole zone list in the account...\n\n"; 

curl -s -X GET "https://api.cloudflare.com/client/v4/zones?page=1&per_page=100&account.id=$accountID" --header 'Content-Type: application/json' --header "X-Auth-Email: $email" --header "X-Auth-Key: $apikey" | jq -r '.result[].id' > zone.list

printf "Now reading hosts registered in Spectrum app...\n"; 

for x in `cat zone.list`; do curl -ks GET "https://api.cloudflare.com/client/v4/zones/$x/spectrum/apps?page=1&per_page=20&direction=desc&order=protocol" --header "X-Auth-Email: $email" --header "X-Auth-Key: $apikey" --header 'Content-Type: application/json' | jq -r '.result[].dns.name' | tee -a spectrumapp.list; done;

printf "\n Now reading IP assigned to each Spectrum app...\n"; 
printf "\n==========================================================================================\n"; 

for dom in `cat spectrumapp.list`; do 
  i=$((i+1));   

  if [ "$sub" != "" ]; then host="$(echo $sub.$dom)"
  else host="$(echo $dom)"
  fi

  ip="$(dig A @1.1.1.1 +short $host.cdn.cloudflare.net | tail -1)"
  if [ "$ip" == "" ]; then edge="$(echo "host does not exist")"
  else edge="$(/usr/bin/whois -h whois.cymru.com $ip | tail -1)"
  fi

  if [ $i -lt 10 ]; then echo $i " ----- " $dom "|" $edge
  elif [ $i -lt 100 ]; then echo $i " ---- " $dom "|" $edge
  elif [ $i -lt 1000 ]; then echo $i " --- " $dom "|" $edge
  fi 

done;

printf "==========================================================================================\n\n";

rm zone.list
rm spectrumapp.list

printf "Job completed.\n";
{% endhighlight %}
