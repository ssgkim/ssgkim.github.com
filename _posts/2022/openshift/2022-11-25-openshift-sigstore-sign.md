---
title: 보안 및 확장성을 위한 샤딩과 sigstore? 
layout: single
author_profile: true
read_time: true
comments: true
share: true
related: true
categories:
- OPENSHIFT
toc: true
toc_sticky: true
toc_label: 목차
description: sigstore의 중요성 
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-11-25 22:00:00 +0800'
---



# 보안 및 확장성을 위한 샤딩

연초부터 보안과 관련된 경고 메시지는 많았습니다. 이를 위한 방법을 어떻게 해결해야 할지 고민한 것들중 컨테이너 원본에 대한 해결책을 고민한 것이 sigstore입니다. sigstore의 투명성을 위한 로그 프로젝트를 포함한 변경 사항이 있습니다. 

[sigstore의 투명성을 위한 로그 Rekor는 최근 v0.6](https://github.com/sigstore/rekor/releases/tag/v0.6.0) 릴리스 에서 중요한 마일드스톤에 도달했으며, 로그 샤딩을 지원합니다. 

로그 샤딩은 단일 Rekor 서버와 연결된 항목을 여러 백엔드 로그에 분산할 수 있음을 의미하며 확장성을 개선하고 [TUF](https://theupdateframework.io/) (The Update Framework ) 표준에 따라 단일 서버에 대한 서명 키의 순환을 가능하게 합니다. sigstore의 1.0 로드맵에서 중요한 단계이며 Red Hatter 및 기타 sigstore 커뮤니티 구성원의 기여와 협업의 결과입니다.

해당 프로젝트에 익숙하지 않은 사람들을 위해 [sigstore](https://next.redhat.com/project/sigstore/) 는 소프트웨어 서명 및 확인을 더 쉽게 만들어 오픈 소스 공급망 보안을 개선하는 것을 목표로 하는 [OpenSSF](https://openssf.org/) 의 프로젝트 입니다. 인증서 투명성과 유사하지만 소프트웨어 서명 자료에 대한 서명 투명성을 위한 Rekor를 포함하여 여러 구성 요소를 포함합니다. Rekor는 소프트웨어 공급망 내에서 생성된 메타데이터의 변조 방지 원장 역할을 합니다. 

공개 원장으로서 Rekor는 특정 다운그레이드 공격과 같은 다양한 위험으로부터 보호하고 개발자와 유지 관리자에게 손상된 키에 대해 경고할 수 있으므로 sigstore의 중요한 부분입니다. [sigstore를 소개하는](https://next.redhat.com/2021/03/09/introducing-sigstore-software-signing-for-the-masses/) 이전 블로그 게시물에서 sigstore에 대한 동기에 대해 자세히 알아 보거나 [sigstore 웹 사이트](https://www.sigstore.dev/) 를 방문하십시오 .

새로운 샤딩 기능은 [공공재 Rekor 인스턴스](https://github.com/sigstore/rekor/#public-instance) 와 다른 Rekor 인스턴스 모두에서 사용할 수 있습니다. 샤딩으로 서버를 실행하기 위한 지침은 [여기에서](https://docs.sigstore.dev/rekor/sharding/) 확인할 수 있습니다 . 로그 샤딩은 선택 사항이며 사용자는 여전히 [샤딩 없이 Rekor 서버를 실행할](https://docs.sigstore.dev/rekor/installation#deploy-a-rekor-server-manually) 수 있습니다 .



# Sigstore Landscape

Sigstore Landscape 는 Sigstore 가 서명한 프로젝트로 가득 차 있으며, FluentBit, Istio, Karpenter, Keptn, Knative, Kubewarden, LinkerD, Pulumi 및 Shipwright와 같은 최신 추가 기능을 확인하면 대세가 확실해 보입니다.

**LLVM:** 이제 Sigstore로 서명하여 사용자가 패키지가 llvm에서 온 것인지 쉽게 확인하고 잠재적인 악성 서명을 감지 Debian/Ubuntu 패키지 용 apt repo 에서 확인

**Updatecli:** 최신 릴리스는 Sigstore로 서명 [Read more](https://www.updatecli.io/)

**Kubernetes release:** 최근 릴리즈 버전인 1.26 Kubernetes 릴리즈는 컨테이너 이미지뿐만 아니라 모든 소프트웨어 아티팩트를 Sigstore로 서명  [Read more](https://venturebeat.com/data-infrastructure/new-kubernetes-1-26-release-boosts-security-storage-teases-dynamic-resource-allocation)



앞으로 확인할 컨텐츠

- [Signatus, ergo securus? Who can sign what with TUF and Sigstore](https://blog.sigstore.dev/signatus-ergo-securus-who-can-sign-what-with-tuf-and-sigstore-ea4d3d84b8b6) by Zack Newman
- [Sigstore Evangelist가 되는 방법은 무엇일까요?](https://blog.sigstore.dev/how-to-become-the-next-sigstore-evangelist-9303ed297e54) by Batuhan Apaydin (developer-guy)
- [Sigstore the easy way](https://rewanthtammana.com/sigstore-the-easy-way/index.html) by Rewanth Tammana
