---
layout: post
title:  "Inspect NS for multiple domains at once"
ref:  20201211a
date:   2020-12-11 18:00:00 +0800
categories: leisure
tags: script dns
lang: en
---

This script will help you when you would like to bulk execute DNS queries, and list nameserver providers of the domains you would like to see.

Firstly, make sure you have `domain.list` in your working directory, separated by enter key, like;

{% highlight ruby %}
firstdomain.com
seconddomain.co.kr
thirddomain.com.au
fourthdomain.com.ph
fifthdomain.com.sg
{% endhighlight %}

Then run this script.

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

Example output:

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

You can see from the result, that `internetsociety.org`, `auda.org.au`, `internetnz.nz`, `irinn.in` use Cloudflare for their DNS.

Caveat: This only reads nameserver information from 1.1.1.1 queried from your own machine and it reads the first line of multiple ns if any. Like e.g.:

{% highlight ruby %}
$ dig NS ripe.net +short
ns3.lacnic.net.
ns3.afrinic.net.
ns4.apnic.net.
manus.authdns.ripe.net.
tinnie.arin.net.
{% endhighlight %}

For some reasons, `ripe.net` uses NS from 5 different regional registries. This script will only catch the first nameserver record and investigate from there, hence it will drop the other NS records. Still how `ripe.net` configured NS should be considered pretty rare.