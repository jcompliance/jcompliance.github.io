---
layout: post
title:  "복수 도메인의 네임서버를 한 번에 검사하기"
ref:  20201211a
date:   2020-12-11 18:00:00 +0800
categories: leisure
tags: script dns
lang: ko
---

여러 개의 검사 대상 도메인 리스트가 있을 때 각각의 도메인에 대한 DNS(네임서버) 공급자 정보를 한 번에 불러오기 위한 스크립트입니다.

먼저, 작업 디렉토리에 `domain.list` 파일을 생성하고 검사 대상 도메인을 엔터 키로 나누어 입력합니다.

{% highlight ruby %}
firstdomain.com
seconddomain.co.kr
thirddomain.com.au
fourthdomain.com.ph
fifthdomain.com.sg
{% endhighlight %}

그리고 아래 스크립트를 돌립니다.

{% highlight ruby %}
#!/bin/bash

i=0; 

echo "\nThis script will read 'domain.list' from the same directory\nand find respective nameserver information of each domain, then list nameserver IP and ASN information.\nThe output will give you an idea about where the nameservers of the domain are currently hosted.";

printf "\n==========================================================================================\n"; 

for dom in `cat domain.list`; do 
  i=$((i+1)); 
  ns="$(dig NS @1.1.1.1 +short $dom | head -1)"
  ip="$(dig A @1.1.1.1 +short $ns | head -1)"
  isp="$(/usr/bin/whois -h whois.cymru.com $ip | tail -1)"

  if [ $i -lt 10 ]; then echo $i " ----- " $dom "|" $ns "|" $isp 
  elif [ $i -lt 100 ]; then echo $i " ---- " $dom "|" $ns "|" $isp 
  elif [ $i -lt 1000 ]; then echo $i " --- " $dom "|" $ns "|" $isp 
  fi 
done;

printf "==========================================================================================\n\n";
{% endhighlight %}

스크립트는 아래와 같은 아웃풋을 보여줄 것입니다.

<pre><nowrap>
This script will read 'domain.list' from the same directory
and find respective nameserver information of each domain, then list nameserver IP and ASN information.
The output will give you an idea about where the nameservers of the domain are currently hosted.

==========================================================================================
1  -----  icann.org | a.icann-servers.net. | 26710 | 199.43.135.53 | ICANN-ANYCASTED-SERVICES, US
2  -----  iana.org | a.iana-servers.net. | 26710 | 199.43.135.53 | ICANN-ANYCASTED-SERVICES, US
3  -----  ripe.net | ns3.lacnic.net. | 28001 | 200.3.13.14 | LACNIC - Latin American and Caribbean IP address, UY
4  -----  apnic.net | ns2.apnic.net. | 18369 | 203.119.95.53 | APNIC-ANYCAST2 APNIC ANYCAST, AU
5  -----  arin.net | u.arin.net. | 42 | 204.61.216.50 | WOODYNET-1, US
6  -----  lacnic.net | a.lactld.org. | 61455 | 200.0.68.10 | LACTLD - LATIN AMERICAN AND CARIBBEAN TLD ASSOCIATION, UY
7  -----  afrinic.net | ns1.afrinic.net. | 33764 | 196.216.2.1 | AFRINIC-ZA-JNB-AS, MU
8  -----  ietf.org | ns0.amsl.com. | 3356 | 4.31.198.40 | LEVEL3, US
9  -----  internetsociety.org | aron.ns.cloudflare.com. | 13335 | 173.245.58.69 | CLOUDFLARENET, US
10  ----  krnic.or.kr | ns0.nic.or.kr. | 9858 | 49.8.14.114 | KRNICNET Korea Internet Security Agency, KR
11  ----  auda.org.au | karl.ns.cloudflare.com. | 13335 | 108.162.193.190 | CLOUDFLARENET, US
12  ----  cnnic.com.cn | b.cnnic.cn. | 24406 | 203.119.26.5 | CNNIC-CRITICAL-AP China Internet Network Infomation Center, CN
13  ----  twnic.tw | dns1.twnic.net.tw. | 3462 | 210.65.47.29 | HINET Data Communication Business Group, TW
14  ----  nic.ad.jp | ns3.nic.ad.jp. | 2515 | 202.12.30.163 | JPNIC Japan Network Information Center, JP
15  ----  sgnic.sg | ps.sgnic.sg. | 9892 | 123.100.252.180 | ICONZ-WEBVISIONS-AP Iconz-Webvisions Pte. Ltd., SG
16  ----  thnic.co.th | a-ns.thnic.co.th. | 17823 | 202.28.1.82 | THNIC-ASN-AP T.H.NIC Co.,Ltd., TH
17  ----  mynic.my | a.mynic.centralnic-dns.com. | 199330 | 194.169.218.114 | CENTRALNIC-ANYCAST-A CentralNic Anycast-A AS Number, GB
18  ----  vnnic.vn | dns1.vnnic.vn. | 23902 | 203.119.73.80 | VNNIC-AS-VN Vietnam Internet network information center (VNNIC), VN
19  ----  internetnz.nz | cf1.ns.internetnz.nz. | 13335 | 162.159.8.104 | CLOUDFLARENET, US
20  ----  apjii.or.id | ns1.apjii.or.id. | 4622 | 203.119.13.18 | ID-NIC Indonesia Network Information Center, ID
21  ----  irinn.in | n1.irinn.in. | 13335 | 162.159.8.208 | CLOUDFLARENET, US
22  ----  cloudflare.com | ns3.cloudflare.com. | 13335 | 162.159.0.33 | CLOUDFLARENET, US
==========================================================================================

</nowrap></pre>

스크립트의 결과로 여러 도메인, `internetsociety.org`, `auda.org.au`, `internetnz.nz`, `irinn.in` 이 Cloudflare를 네임서버 공급자로 이용한다는 것을 알 수 있습니다.

이 스크립트는 사용하는 기기에서 1.1.1.1로 DNS를 쿼리하며, 복수의 네임서버 정보 중 가장 첫번째 라인만 읽는다는 점에 주의하십시오. 따라서:

{% highlight ruby %}
$ dig NS ripe.net +short
ns3.lacnic.net.
ns3.afrinic.net.
ns4.apnic.net.
manus.authdns.ripe.net.
tinnie.arin.net.
{% endhighlight %}

위의 `ripe.net` 처럼 네임서버를 5개의 각기 다른 공급자를 이용하는 드문 경우가 있다면 스크립트에서는 가장 첫 번째 줄만 읽어오고 나머지 결과는 버릴 것입니다. 다만 네임서버에 있어 프라이머리-세컨더리 구조 대신 모든 다른 공급자를 한번에 노출시키는 유즈케이스는 흔하지 않습니다.