---
layout: post
title:  "Cloudflare 기술 세일즈 기본 과정"
ref:  20200615a
date:   2020-06-15 15:00:00 +0800
categories: cloudflare overview
tags: cloudflare overview partner performance security
lang: ko
---

Cloudflare 파트너 세일즈 엔지니어분들의 교육을 위한 영상자료를 촬영했습니다. 'Selling Cloudflare - Technical Overview' 라는 주제입니다. 2시간 분량으로 아래 내용을 국문으로 설명하는 비디오입니다. 파트너사 전용으로 포털을 통해 배포됩니다.

- Cloudflare와의 파트너십의 이점
- Cloudflare의 하이 레벨 기술 구조 `// part 1`
- Cloudflare 제품 오버뷰
- 코어 제품 요약
   - DNS
   - TLS/SSL `// part 2`
   - Security
   - Performance
   - Platform
   - Insights `// part 3`
- Cloudflare 구현 과정 오버뷰 & POC 주의 사항
- Add-on 제품 요약
   - Cloudflare for Teams (Access, Gateway)
   - Magic Transit
   - Spectrum
   - Load Balancing
   - Stream, Stream Delivery
   - Argo
      - Argo Smart Routing
      - Argo Tiered Caching
      - Argo Tunnel
   - Workers
   - Rate Limiting
   - Bot Management
   - DNS Firewall, Secondary DNS
   - SSL for SaaS
   - Access/Mutual TLS
   - Registrar
   - Image Resizing
- 주로 참고하실 리소스 `// part 4`

<!--
2시간이 훌쩍 넘는 비디오라 쉬는 시간 없이 한 번에 달리시면 자장가가 될까 염려되어 (시원스쿨의 스킬을 참고하여) 30분정도씩 잘랐습니다. 구현/설정까지는 다루지 않지만 Cloudflare 세일즈를 시작하실 때 필요한 기본 지식을 최대한 다 설명드린다는 생각으로 촬영했습니다. 비디오와 자료는 파트너사에만 배포가 가능하기에, Cloudflare 리셀러/Tier-1 파트너께서는 발급받으신 계정으로 파트너 포털에 접속하셔서 학습하실 수 있습니다.

파트너사 엔지니어분들께서는 해당 모듈을 학습하시고, [Cloudflare Accredited Sales Engineer 시험에 응시해 보세요](https://blog.cloudflare.com/empowering-our-customers-and-service-partners/). 합격하시면 다소 기분이 좋아지는 인증서와 부상이 주어집니다. 저희 회사도 언젠가는 시스코처럼 풍요로운 자격증 비즈니스(?)가 생겼으면 하는 꿈을 제가 개인적으로 늘 꾸고 있답니다. 아직 프로그램 초창기라 따기 어렵지 않으니, 사냥하세요!

!["ASE" 티셔츠를 입고 있으면... 남편이 얼마나 놀리는지 모릅니다](https://blog-cloudflare-com-assets.storage.googleapis.com/2020/04/image-2.png)

고객사 용으로도 국문 비디오나 국문 자료를 더 만들고 싶은 마음은 늘 한결같은데, 하루가 24시간밖에 없어 생각처럼 못하고 있습니다. 만일 저희 고객사 담당자분이시고 이 글을 우연히 발견하셨다면 이 비디오는 고객사에 나가는 자료는 아니니 너른 이해 부탁드립니다. 그러나 나열된 특정 제품이나 주제에 대해 궁금하신 경우, 엔터프라이즈 계정 담당 팀에게 문의해 주시면 제 동료분들께서 친절하게 도와주실 것입니다.
-->

# 2020-06-26 Update

이곳은 공식 포털이 아닙니다만, 파트너 포털에 교육자료 업로드 내부 승인을 받는 데에 시간이 걸리고 있어, 당장 영상이 필요하신 파트너 엔지니어분들을 지원해 드리기 위해 임시적으로 블로그에 영상을 업로드합니다. 파트너 포털에 영상이 준비되고 나면 블로그에서는 내려갑니다. 학습에 도움이 되시길 바랍니다.

<!--
## Part 1

<stream src="642b534cb3fef1aeb9defec30cf74bb8" controls preload></stream>
<script data-cfasync="false" defer type="text/javascript" src="https://embed.videodelivery.net/embed/r4xu.fla9.latest.js?video=642b534cb3fef1aeb9defec30cf74bb8"></script>

Cloudflare와의 파트너십의 이점 / 솔루션의 이점 / 기술구조의 개요를 다룹니다. (21분)

## Part 2

<stream src="fde8bf6a3d4d49f3baa7951b5190a3bf" controls preload></stream>
<script data-cfasync="false" defer type="text/javascript" src="https://embed.videodelivery.net/embed/r4xu.fla9.latest.js?video=fde8bf6a3d4d49f3baa7951b5190a3bf"></script>

Cloudflare 솔루션을 하이 레벨로 짚어보고, 코어 제품 중 DNS와 SSL을 설명합니다. (31분)

## Part 3

<stream src="2c53aecfd09a384450479096718bdfe8" controls preload></stream>
<script data-cfasync="false" defer type="text/javascript" src="https://embed.videodelivery.net/embed/r4xu.fla9.latest.js?video=2c53aecfd09a384450479096718bdfe8"></script>

Cloudflare 코어 제품 중 Security, Performance, Platform, Insight 부분을 짚어봅니다. (32분)

## Part 4

<stream src="892c0d8a88e7913e27277cc7828b8247" controls preload></stream>
<script data-cfasync="false" defer type="text/javascript" src="https://embed.videodelivery.net/embed/r4xu.fla9.latest.js?video=892c0d8a88e7913e27277cc7828b8247"></script>

Cloudflare 구현 과정을 하이 레벨로 짚어보고, POC 제안 시 주의 사항을 알아보고 여러가지 Add-on 솔루션을 빠르게 다룹니다. (29분) 


이번 코스워크의 목적은 파트너사 엔지니어분들이 직접 고객사에 Cloudflare 솔루션을 소개하시고, 고객사의 요구사항에 따라 솔루션을 디자인하시고 high level로 scoping 하실 수 있게 되는 것입니다. 

이 과정을 학습하신 후 Partner Demo Account를 발급받으셔서 실제 솔루션을 hands-on 테스트해보시는 게 좋습니다. 직접 동작을 테스트해보시고, 고객사에 Dashboard Demo를 직접 하실 수 있게 되시면 그 후에는 실제 고객사의 POC 지원 & implementation 과정을 학습하실 수 있게 됩니다. Tier-1 파트너분들께서는 최종적으로 troubleshooting 까지 익히셔서, Cloudflare SE의 지원 없이도 직접 Tier-1으로 담당 고객사의 기술지원을 하실 수 있도록 될 것입니다.

그 시작으로, hands-on 테스트를 시작해보시는 것을 추천합니다. 참고하실 만한 리소스를 아래 소개합니다.

- [Full Setup 설명 영상(국문)](/cloudflare/onboarding/2020/04/20/hello-world-ko.html)
- [CNAME Setup 설명 영상(국문)](https://youtu.be/PEVbptIL38U)
- [Cloudflare 테스트 시작하기(공식/영문)](https://support.cloudflare.com/hc/en-us/articles/360037345072-Getting-Started-with-Cloudflare-Video-Tutorials)

해당 모듈을 학습하시고 나면 [Cloudflare Accredited Sales Engineer 시험에 응시하실 수 있습니다](https://blog.cloudflare.com/empowering-our-customers-and-service-partners/). 담당 Channel Account Manager 분께 문의하시기 바랍니다.

!["ASE" 티셔츠를 입고 있으면... 남편이 얼마나 놀리는지 모릅니다](https://blog-cloudflare-com-assets.storage.googleapis.com/2020/04/image-2.png)

-->

# 2020-10-16 Update

드디어 파트너 포털에 국문 영상이 업로드 완료되어 블로그에서 비디오를 내렸습니다. Cloudflare 파트너 포털에 접속하셔서 해당 코스워크를 학습하시기 바랍니다. Cloudflare University - Accreditations - Library 에서 찾으실 수 있습니다.

4개로 나누어 촬영한 트레이닝 비디오의 각각 파트는 아래 내용을 다루고 있습니다. 1개의 코스워크입니다만 집중력 저하를 우려하여 30분 정도씩 잘랐습니다.

- Part 1 : Cloudflare와의 파트너십의 이점 / 솔루션의 이점 / 기술구조의 개요를 다룹니다. (21분)
- Part 2 : Cloudflare 솔루션을 하이 레벨로 짚어보고, 코어 제품 중 DNS와 SSL을 설명합니다. (31분)
- Part 3 : Cloudflare 코어 제품 중 Security, Performance, Platform, Insight 부분을 짚어봅니다. (32분)
- Part 4 : Cloudflare 구현 과정을 하이 레벨로 짚어보고, POC 제안 시 주의 사항을 알아보고 여러가지 Add-on 솔루션을 빠르게 다룹니다. (29분) 

파트너사 세일즈 엔지니어분께서 4개 트레이닝 비디오의 시청이 끝나셨으면 실제 Cloudflare 구현을 테스트해보시기 바랍니다. 아래 리소스를 참고하실 수 있습니다.

- [Full Setup 설명 영상(국문)](/cloudflare/onboarding/2020/04/20/hello-world-ko.html)
- [CNAME Setup 설명 영상(국문)](https://youtu.be/PEVbptIL38U)
- [Cloudflare 테스트 시작하기(공식/영문)](https://support.cloudflare.com/hc/en-us/articles/360037345072-Getting-Started-with-Cloudflare-Video-Tutorials)

해당 모듈의 학습과 테스트가 끝나셨으면 아래 Certification 중 Cloudflare ASP(Accredited Sales Professional) 과 Cloudflare ASE(Accredited Sales Engineer) 시험에 응시하실 수 있습니다. 파트너 포털에서 시험에 도전하시기 바랍니다.

!["ASE" 티셔츠를 입고 있으면... 남편이 얼마나 놀리는지 모릅니다](https://blog-cloudflare-com-assets.storage.googleapis.com/2020/04/image-2.png)