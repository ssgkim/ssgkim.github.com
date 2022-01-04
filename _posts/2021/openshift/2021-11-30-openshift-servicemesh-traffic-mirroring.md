---
title: OpenShift Service Mesh의 트래픽 미러링
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
description: Service Mesh의 트래픽 미러링
article_tag1: OpenShift
article_tag2: ServiceMesh
article_tag3: infra
article_section: IT근황 공유하기
meta_keywords: openshift,kubernetes,cncf,infra,servicemesh
last_modified_at: '2021-11-30 23:00:00 +0800'
---

안전하고 예측 가능한 모드에서 라이브 프로덕션으로 테스트 사례를 수행하는 방법은? Service Mesh 내에서 어떻게 이를 달성할 수 있을까? 
트래픽 미러링이란 무엇이며 이 사례를 어떻게 활용할까?

메쉬 인하자!!

## 개요

비프로덕션/테스트 환경에서 서비스를 테스트하기 위해 가능한 모든 테스트 케이스 조합을 열거하는 것은 어렵고 위험할 수 있습니다. 어떤 경우에는 이러한 사용 사례를 카탈로그화하는 데 드는 모든 노력이 실제 프로덕션 사용 사례와 일치하지 않는다는 것을 알게 될 것입니다. 이상적인 시나리오에서는 실제 프로덕션 사용 사례와 트래픽을 사용하여 보다 인위적인 테스트 환경에서 놓칠 수 있는 테스트 중인 서비스의 모든 기능 영역을 밝힐 수 있습니다.

이 블로그 게시물에서는 트래픽 미러링 배포를 분석하고 Service Mesh에서 구현하는 방법을 설명합니다.

참고: 이 블로그 게시물은 Github 에 있는 [istio-files 저장소](https://github.com/rcarrata/istio-files) 에서 지원됩니다.

## 전제 조건

- OpenShift 4.x 클러스터(4.3+ 클러스터에서 테스트됨)
- OpenShift Service Mesh Operators 설치(이 블로그 게시물의 v1.1.1)
- 서비스 메시 컨트롤 플레인 배포
- 배포된 4개의 마이크로서비스(두 번째 블로그 게시물 참조)
- Service Mesh 내에 추가된 마이크로서비스(세 번째 블로그 게시물 참조)
- Service Mesh에 마이크로서비스 포함(네 번째 블로그 게시물 참조)

클러스터 및 네임스페이스를 식별하기 위해 이 환경 변수를 내보냅니다.

```
$ export APP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $OCP_SUBDOMAIN
apps.ocp4.rglab.com
export NAMESPACE="istio-tutorial"
```

## 1. 트래픽 미러링

섀도잉이라고도 하는 트래픽 미러링은 기능 팀이 가능한 한 적은 위험으로 프로덕션에 변경 사항을 가져올 수 있는 강력한 개념입니다. 미러링은 라이브 트래픽의 복사본을 미러링된 서비스로 보냅니다. 미러링된 트래픽은 기본 서비스에 대한 중요 요청 경로의 대역 외에서 발생합니다.

이를 위해 고객 v1에서 고객 v2로 VirtualService의 트래픽을 미러링하는 접근 방식이 있을 수 있습니다.

```
http:
- route:
  - destination:
      host: customer
      subset: version-v1
  mirror:
    host: customer
    subset: version-v2
```

트래픽 미러 가상 서비스 렌더링 및 적용

```
$ cat istio-files/customer-mirror-traffic.yml | envsubst | oc apply -f -
Warning: oc apply should be used on resource created by either oc create --save-config or oc apply
virtualservice.networking.istio.io/customer configured
destinationrule.networking.istio.io/customer unchanged
```

고객 경로에 요청:

```
$ echo $customer_route
customer-istio-tutorial-istio-system.apps.ocp4.rglab.com

$ curl $customer_route
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 608
```

경로가 ingressgateway에서 고객 v1으로 라우팅될 것으로 예상했지만 카펫 뒤에서도 흥미로운 일이 일어나고 있습니다.

미러링된 트래픽에 어떤 일이 발생했는지 보려면 istio-proxy 로그를 확인하세요.

```
$ oc logs -f customer-v2-2-d8z45 -c istio-proxy
...
[2020-06-05T10:59:15.857Z] "GET / HTTP/1.1" 200 - "-" "-" 0 69 46 43 "192.168.7.77,10.254.3.1,10.254.3.16" "curl/7.69.1" "aa394a0c-464f-9cf0-a987-592fa0d2b0a0" "customer-istio-tutorial-istio-system.apps.ocp4.rglab.com-shadow" "127.0.0.1:8080" inbound|8080|http|customer.istio-tutorial.svc.cluster.local - 10.254.3.24:8080 10.254.3.16:0 outbound_.8080_.version-v2_.customer.istio-tutorial.svc.cluster.local default
```

요청이 customerv2에도 도달하고 있지만 응답이 클라이언트로 전송되지 않는다는 것을 알아차렸습니다(이 경우 컬). 이는 요청된 호스트에 "-shadow" 플래그가 있기 때문입니다.

결론적으로 이 라우팅 규칙은 트래픽의 100%를 v1으로 보냅니다. 마지막 스탠자는 customer:v2 서비스에 미러링할 것임을 지정합니다. 트래픽이 미러링되면 요청은 -shadow가 추가된 Host/Authority 헤더와 함께 미러링된 서비스로 전송됩니다. 예를 들어 cluster-1은 cluster-1-shadow가 됩니다.

또한 이러한 요청은 "실행 후 잊어버리기"로 미러링되므로 응답이 삭제된다는 점에 유의하는 것이 중요합니다.

kiali에서 이 미러 트래픽이 표시되는 이유는 고객 v1이 클라이언트에서 요청한 것을 보여주지만 customerv2도 이 트래픽 미러를 생성하기 때문에 기본 설정 서비스로 트래픽을 보내고 있기 때문입니다.

[![Istio 트래픽 미러링](https://rcarrata.com/images/istio7.png)](https://rcarrata.com/images/istio7.png)

### 미러 요청 변경

또한 모든 요청을 미러링하는 대신 mirror_percent 필드를 사용하여 트래픽의 일부를 미러링할 수 있습니다. 이 필드가 없으면 이전 버전과의 호환성을 위해 모든 트래픽이 미러링됩니다.

```
http:
- mirror:
    host: customer
    subset: version-v2
  mirror_percent: 50
  route:
  - destination:
      host: customer
      subset: version-v1
    weight: 100
```

mirror_request의 50%와 함께 수정된 mirror_percent를 적용합니다.

```
$ cat istio-files/customer-mirror-traffic-adv.yml | envsubst | oc apply -f -
virtualservice.networking.istio.io/customer configured
destinationrule.networking.istio.io/customer unchanged
```

해피메싱!!
