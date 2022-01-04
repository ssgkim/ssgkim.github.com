---
title: OpenShift Service Mesh의 수신 라우팅 및 트래픽 관리
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
description: Service Mesh의 수신 라우팅 및 트래픽 관리
article_tag1: OpenShift
article_tag2: logging
article_tag3: infra
article_section: IT근황 공유하기
meta_keywords: openshift,kubernetes,cncf,infra,network
last_modified_at: '2021-11-16 23:00:00 +0800'
---

Service Mesh에서 Ingress 수신 라우팅을 구성하는 방법과 관련된 구성 요소는? 그리고 Service Mesh에서 OpenShift Routes와 Ingress 라우팅 간의 주요 차이점은?

메쉬 인하자!!

## 개요

이 블로그 게시물에서는 트래픽 관리, Service Mesh의 수신 라우팅 및 Service Mesh 내에 배포된 애플리케이션으로 트래픽을 가져오는 데 관련된 구성 요소에 대해 자세히 설명합니다.

참고: 이 블로그 게시물은 Github 에 있는 [istio-files 저장소](https://github.com/rcarrata/istio-files) 에서 지원됩니다.

## 0. 전제 조건

- OpenShift 4.x 클러스터(4.3+ 클러스터에서 테스트됨)
- OpenShift Service Mesh Operators 설치(이 블로그 게시물의 v1.1.1)
- 서비스 메시 컨트롤 플레인 배포
- 배포된 4개의 마이크로서비스
- Service Mesh 내에 추가된 마이크로서비스

## 1. 기존 K8S 서비스 조정

OpenShift에서 oc new-app을 사용하면 이 명령 내에서 여러 kubernetes 리소스가 생성됩니다. 이 서비스 중 하나가 서비스입니다.

배포된 고객 마이크로서비스의 서비스를 확인하십시오.

```
 oc get svc/customer -n $OCP_NS -o json | jq -r '[.spec.selector]'
[
  {
    "app": "customer",
    "deploymentconfig": "customer",
    "version": "v1"
  }
]
```

우리가 알고 있듯이 kubernetes 서비스는 우리가 정의한 선택기를 기반으로 포드와 일치합니다. 이전 예제에서 앱, deploymentconfig 및 버전 레이블은 이 팟(Pod)과의 일치를 정의합니다.

따라서 이는 해당 레이블과 일치하는 팟(Pod)만 이 특정 서비스의 로드 밸런싱을 사용한다는 것을 의미합니다(서비스 '뒤에' 있음).

그러나 Service Mesh에서는 다음 섹션에서 설명할 대상 규칙으로 특정 버전을 구별할 것이기 때문에 좀 더 넓은 선택기에서 서비스를 정의해야 합니다.

새 선택기의 예는 다음과 같습니다.

```
cat istio-files/customer/kubernetes/Service.yml | grep selector -A5
 selector:
   app: customer
```

이 선택기는 레이블이 app: customer인 모든 포드와 일치하며 이 가능성에는 동일한 애플리케이션의 여러 버전이 포함됩니다.

특정 명령을 실행하여 기존 oc new-app 서비스를 삭제하고 istio에 적용된 서비스를 적용합니다.

```
$ oc delete svc/customer -n $OCP_NS
service "customer" deleted

$ oc delete svc/preference -n $OCP_NS
service "preference" deleted

$ oc delete svc/recommendation -n $OCP_NS
service "recommendation" deleted

$ oc delete svc/partner -n $OCP_NS
service "partner" deleted
$ oc apply -f istio-files/customer/kubernetes/Service.yml -n $OCP_NS
service/customer created

$ oc apply -f istio-files/preference/kub
ernetes/Service.yml -n $OCP_NS service/preference created

$ oc apply -f istio-files/recommendation/kubernetes/Service.yml -n $OCP_NS
service/recommendation created

$ oc apply -f istio-files/partner/kubernetes/Service.yml -n $OCP_NS
service/partner created
```

------

## 2. 인그레스 라우팅 활성화

이제 적절한 서비스 패치가 있으므로 Mesh 및 OpenShift 클러스터 외부의 마이크로 서비스에 도달하고 사용하기 위해 수신 라우팅을 활성화해야 합니다.

### 2.1 OpenShift 경로 대 수신 서비스 메시

OpenShift 경로의 수신과 Service Mesh를 사용한 수신 라우팅 사이에는 주요 차이점이 있습니다.

OpenShift Route는 트래픽을 클러스터로 가져오기 위해 OpenShift Ingresscontroller/Router(Haproxy)를 사용하고 라우터는 레이블이 있는 특정 서비스를 가리키며 해당 서비스를 통해 애플리케이션의 특정 포드에 도달합니다.

- 기본적으로 OpenShift 경로 수신:

```
Route (HAProxy/Router) => SVC1 (ip-pod-1, ip-pod-2, ...)
```

지금까지는 너무 좋죠? 그러나 Service Mesh 내에서 이 경로를 사용하면 외부에 노출된 서비스에서 Service Mesh의 기능을 놓칠 수 있습니다. 일부 정책 및 규칙이 클라이언트 프록시에 적용되기 때문입니다.

따라서 수신 라우팅에서 Istio의 기능을 활용하기 위해 다음 Istio 리소스를 사용합니다.

- 인그레스 게이트웨이
- 가상 서비스
- 대상 규칙

이 구성 요소에 대해 잠시 설명하겠지만 지금은 높은 수준에서 Mesh 내의 인그레스 라우팅이 다음과 같습니다.

- Service Mesh 내 수신 라우팅

```
Route (HAProxy/Router) => istio-ingressgateway  => SVC1 (ip-pod-1, ip-pod-2, ...)
```

따라서 트래픽을 클러스터와 Service Mesh로 가져오면 수신 게이트웨이가 사용되지만 동일한 수신 게이트웨이를 통해 액세스하는 대신(및 /customer 또는 /partner와 같은 여러 경로 사용) 다음을 가리키는 경로를 사용하고 있습니다. istio 인그레스 게이트웨이.

인그레스 게이트웨이를 정의하고 VirtualService를 사용하면 요청이 이 인그레스 게이트웨이를 통해 k8 서비스로 라우팅되므로 최종적으로 Pod로 라우팅됩니다.

아주 쉽지는 않지만 실제로는 이렇게 보이는 것이 더 간단합니다. 이 과정에 사용된 구성 요소를 살펴보겠습니다.

### 2.2 트래픽 관리를 위한 Service Mesh의 구성 요소

- **가상 서비스는** 당신이 요청 Istio 및 플랫폼에서 제공하는 기본 연결과 발견에 구축, Istio 서비스 메쉬 내에서 서비스로 라우팅하는 방법을 구성 할 수 있습니다.
- 게이트웨이를 사용하는 VirtualService의 예1(이 경우에는 customer-gw):

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customer
  namespace: ${NAMESPACE}
spec:
  gateways:
  - customer-gw
  hosts:
  - customer-${NAMESPACE}-istio-system.${APP_SUBDOMAIN}
  http:
  - route:
    - destination:
        host: customer
        subset: version-v1
      weight: 100
```

- 트래픽을 분리하기 위해 사용자 헤더를 사용하는 VirtualService의 예 2:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

- **대상 규칙** 은 가상 서비스 라우팅 규칙이 평가된 후에 적용되므로 트래픽의 "실제" 대상에 적용됩니다.

지정된 서비스의 모든 인스턴스를 버전별로 그룹화하는 것과 같이 대상 규칙을 사용하여 명명된 서비스 하위 집합을 지정합니다. 그런 다음 가상 서비스의 라우팅 규칙에서 이러한 서비스 하위 집합을 사용하여 서비스의 다른 인스턴스에 대한 트래픽을 제어할 수 있습니다.

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

실제로 **가상 서비스는 트래픽을 지정된 대상으로 라우팅** 한 다음 **대상 규칙을** 사용 **하여 해당 대상의 트래픽에 어떤 일이 발생하는지 구성하는 것으로 생각하십시오** .

- 게이트웨이: 게이트웨이를 사용하여 메시에 대한 인바운드 및 아웃바운드 트래픽을 관리하여 메시에 들어오거나 나갈 트래픽을 지정할 수 있습니다.

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

### 2.3 마이크로서비스를 위한 서비스 메시 객체 생성

수신 라우팅을 사용하여 마이크로 서비스를 노출하려면 이전 섹션에서 검토한 일부 객체를 생성해야 합니다.

이 istio 객체를 적용한 후에는 이전 경로가 더 이상 작동하지 않고 새 경로가 더 이상 마이크로서비스의 네임스페이스에 없을 것이며 모든 경로와 수신 게이트웨이는 istio-system 네임스페이스에 위치하게 됩니다.

따라서 우리의 경우 적용될 객체는 다음과 같습니다.

```
Route (HAProxy/Router) => istio-ingressgateway  => SVC1 (ip-pod-1, ip-pod-2, ...)
```

- 경로(ns => istio-system)
- 게이트웨이(ns => istio-tutorial)
- VirtualService(ns => istio-tutorial)
- DestinationRule(ns => istio-tutorial) - MTLS 활성화

수신 라우팅을 활성화하기 위해 객체를 렌더링하고 적용합니다.

```
$ export APP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $APP_SUBDOMAIN
apps.ocp4.rglab.com
export OCP_NS="istio-tutorial"
$ cat istio-files/customer-ingress_mtls.yml | NAMESPACE=$(echo $OCP_NS) envsubst | oc apply -f -
route.route.openshift.io/customer created
gateway.networking.istio.io/customer-gw created
virtualservice.networking.istio.io/customer created
destinationrule.networking.istio.io/customer created
```

### 2.4 인그레스 라우팅 객체를 자세히 살펴보겠습니다.

먼저 경로:

```
$ oc get route -n istio-system customer -o yaml --export
...
spec:
  host: customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
  port:
    targetPort: http2
  to:
    kind: Service
    name: istio-ingressgateway
    weight: 100
  wildcardPolicy: None
status:
  ingress: null
```

경로가 {app}{app-ns}{istio-ns}.apps의 호스트와 http2용 포트를 사용하는 istio-ingressgateway 서비스를 가리키는 것을 볼 수 있습니다.

게이트웨이의 경우 customer-gw는 앱의 네임스페이스에 정의됩니다.

```
$ oc get gateway -n istio-tutorial customer-gw -o yaml | yq .spec
{
  "selector": {
    "istio": "ingressgateway"
  },
  "servers": [
    {
      "hosts": [
        "customer-istio-tutorial-istio-system.apps.ocp4.rglab.com"
      ],
      "port": {
        "name": "http2",
        "number": 80,
        "protocol": "HTTP2"
      }
    }
  ]
}
```

우리가 볼 수 있듯이 선택자에는 istio: ingressgateway가 있고 istio ingressgateway와 연결되어 있고 경로에 호스트와 포트도 정의되어 있습니다.

```
$ oc get virtualservice -n istio-tutorial customer -o yaml | yq .spec
{
  "gateways": [
    "customer-gw"
  ],
  "hosts": [
    "customer-istio-tutorial-istio-system.apps.ocp4.rglab.com"
  ],
  "http": [
      "route": [
        {
          "destination": {
            "host": "customer",
            "subset": "version-v1"
          }
        }
      ]
    }
  },
}
```

가상 서비스는 customer-gw의 게이트웨이에 대한 포인트를 사용하고 고객 호스트 호스트의 하위 집합과 version-v1의 하위 집합으로 경로 요청을 설정합니다.

마지막으로 DestinationRule은 다음과 같이 정의됩니다.

```
$ oc get destinationrule -n istio-tutorial customer -o yaml | yq .spec
{
  "host": "customer",
  "subsets": [
    {
      "labels": {
        "version": "v1"
      },
      "name": "version-v1"
    },
  "trafficPolicy": {
    "tls": {
      "mode": "ISTIO_MUTUAL"
    }
  }
}
```

대상 규칙은 현재 보유하고 있는 버전의 하위 집합(실제로는 버전-v1이지만 다음 실습에서는 확장될 예정임)을 설정합니다. 또한 이 대상 규칙에 속하는 호스트와 중요한 기능: 상호 TLS를 활성화합니다.

다른 블로그 게시물은 상호 TLS를 심층적으로 분석하는 데 전념하지만 이 트래픽 정책으로 MTLS가 활성화되어 있는지 확인하십시오.

## 3. Service Mesh에서 마이크로서비스의 인그레스 라우팅 테스트

istio-system 네임스페이스에서 생성된 경로를 확인합니다.

```
$ oc get route -n istio-system customer
NAME       HOST/PORT                                                  PATH   SERVICES               PORT    TERMINATION   WILDCARD
customer   customer-istio-tutorial-istio-system.apps.ocp4.rglab.com          istio-ingressgateway   http2                 None
```

모든 것이 정상인지 테스트하기 위해 컬링하십시오.

```
$ curl customer-istio-tutorial-istio-system.apps.ocp4.rglab.com -I
HTTP/1.1 200 OK
content-type: text/plain;charset=UTF-8
content-length: 0
```

효과가있다!

Kiali로 트래픽 흐름을 확인하십시오.

[![Kiali 단순 트래픽 관리](https://rcarrata.com/images/istio1.png)](https://rcarrata.com/images/istio1.png)

중요: 이전 경로는 더 이상 작동하지 않습니다. istio-sytem에서 모든 새 경로를 사용하세요.

```
$ oc get route -n istio-tutorial customer
NAME       HOST/PORT                                     PATH   SERVICES   PORT       TERMINATION   WILDCARD
customer   customer-istio-tutorial.apps.ocp4.rglab.com          customer   8080-tcp                 None

$ curl -I customer-istio-tutorial.apps.ocp4.rglab.com
HTTP/1.0 503 Service Unavailable
```

## 4. 파트너 마이크로서비스에 istio 라우팅 적용

이제 모든 것이 고객 마이크로서비스 인그레스 라우팅과 함께 작동한다는 것을 알았으므로 이를 파트너 마이크로서비스에 적용해 보겠습니다.

```
$ cat istio-files/partner-ingress_mtls.yml | NAMESPACE=$(echo $OCP_NS) envsubst | oc apply -f -
route.route.openshift.io/partner created
gateway.networking.istio.io/partner-gw created
virtualservice.networking.istio.io/partner created
destinationrule.networking.istio.io/partner created
```

모든 것이 정상인지 확인하기 위해 테스트합니다.

```
$ curl -I partner-istio-tutorial-istio-system.apps.ocp4.rglab.com
HTTP/1.1 200 OK
x-application-context: application
content-type: text/plain;charset=UTF-8
content-length: 76

$ curl partner-istio-tutorial-istio-system.apps.ocp4.rglab.com
partner => preference => recommendation v1 from 'recommendation-2-kjpsc': 9
```

이제 인그레스 라우팅 시스템을 위한 두 가지 경로가 있습니다.

```
$ oc get routes -n istio-system | egrep "customer|partner"
customer               customer-istio-tutorial-istio-system.apps.ocp4.rglab.com          istio-ingressgateway   http2                        None
partner                partner-istio-tutorial-istio-system.apps.ocp4.rglab.com           istio-ingressgateway   http2                        None
```
해피 서비스 메싱!
