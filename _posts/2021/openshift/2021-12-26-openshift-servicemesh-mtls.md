---
title: OpenShift Service Mesh의 제로 트러스트 네트워크 및 mTLS 심층 분석
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
description: 마이크로 서비스 간의 트래픽이 제로 트러스트 네트워크에서 암호화되고 안전하다는 것을 어떻게 보장할 수 있을까요?
article_tag1: OpenShift
article_tag2: ServiceMesh
article_tag3: infra
article_section: IT근황 공유하기
meta_keywords: openshift,kubernetes,cncf,infra,servicemesh
last_modified_at: '2021-12-26 23:00:00 +0800'
---


마이크로 서비스 간의 트래픽이 제로 트러스트 네트워크에서 암호화되고 안전하다는 것을 어떻게 보장할 수 있을까? Service Mesh 클러스터에서 mTLS를 쉽게 활성화할 수 있는 방법은 무엇일까? 그리고 안전한 마이크로서비스 배포를 위해 다양한 CR 및 Istio 개체를 관리하는 방법을 어떻게 확인할 수 있을까요?

메쉬 인하자!

이것은 OpenShift의 Service Mesh 시리즈의 여덟 번째 블로그 게시물입니다. 이전 게시물 확인:


## 가. 개요

상호 인증 또는 양방향 인증은 두 당사자가 동시에 서로를 인증하는 것을 말하며, 일부 프로토콜(IKE, SSH)에서는 기본 인증 모드이고 다른 프로토콜(TLS)에서는 선택적 인증 모드입니다.

Red Hat OpenShift Service Mesh를 사용하면 상호 TLS가 발생하고 있다는 사실을 애플리케이션/서비스 없이도 사용할 수 있습니다. TLS는 서비스 메시 인프라와 두 사이드카 프록시 간에 완전히 처리됩니다.

[![mtls 99](https://rcarrata.com/images/mesh.png)](https://rcarrata.com/images/mesh.png)

mTLS Istio 기능은 클러스터 수준 또는 네임스페이스 수준에서 활성화할 수 있습니다. 네임스페이스 수준에서 활성화하여 mTLS 보안을 제어하는 Istio 개체를 시연합니다.

이 mtls 심층 분석 블로그 게시물의 모든 테스트는 다음에서 실행됩니다.

- OpenShift 4.7
- Servicemesh operator 2.0.8
- jaeger 1.24.1
- elasticsearch 5.2.2
- kiali 1.24.0

참고: 이 블로그 게시물은 내 개인 Github 에 있는 [istio-files 저장소](https://github.com/rcarrata/istio-files/tree/mesh-v2) 에서 지원합니다 .

## B. Service Mesh Control Operators 및 ControlPlane 설치

이 블로그 게시물은 이전 실습에서 사용된 이전 버전의 Service Mesh v1과 약간 다른 Istio 1.6 기반 Service Mesh v2를 설치할 것이기 때문에 OpenShift "빈"(새로 설치됨)에서 시작합니다.

- 서비스 메시 운영자 설치

```
helm install service-mesh-operators -n openshift-operators service-mesh-operators/
```

- helm ls를 사용하여 Service Mesh Operators 상태를 확인합니다.

```
helm ls -n openshift-operators
```

- OpenShift 운영자 팟(Pod)의 상태도 확인하십시오.

```
oc get pod -n openshift-operators
```

- 그런 다음 Service Mesh 컨트롤 플레인을 설치합니다.

```
export deploy_namespace=istio-system

oc new-project ${deploy_namespace}

#Note: you may need to wait a moment for the operators to propagate

helm install control-plane -n ${deploy_namespace} control-plane/
```

- 서비스 메시 컨트롤 플레인 모니터링

```
oc get pod -n istio-system
NAME                                    READY   STATUS              RESTARTS   AGE
grafana-58b8d6b866-vkzfz                2/2     Running             0          59s
istio-egressgateway-74688d758-n9kcw     1/1     Running             0          59s
istio-ingressgateway-566b8799f5-tvmtv   1/1     Running             0          59s
istiod-basic-install-7fd5988b8d-7h2s6   1/1     Running             0          116s
jaeger-5bb5848ff5-vg927                 2/2     Running             0          59s
kiali-75d59c6544-2r5qp                  0/1     ContainerCreating   0          11s
prometheus-854f88478f-mqlq4             3/3     Running             0          88s
```

### 나.1. 배포된 ServiceMeshControlPlane 개체를 분석합니다.

ServiceMeshControlPlane은 각 Service Mesh 컨트롤 플레인의 설정 및 구성 요소를 제어하는 Service Mesh CR입니다. 우리의 경우 Service Mesh Operator의 간섭 없이 이 구성 요소를 수동으로 조정할 것이기 때문에 보안 섹션에서 mtls, automtls 및 controlPlane mtls를 비활성화합니다.

```
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: 
  namespace: 
spec:
...
  security:
    dataPlane:
      mtls: false
      automtls: false
    controlPlane:
      mtls: false
```

체크 [전체 서비스 메쉬 제어 평면 CR 객체](https://github.com/rcarrata/istio-files/blob/mesh-v2/control-plane/templates/istio-controlplane.yaml) 가 자세한 내용은이 깊은 다이빙 블로그 게시물에 사용되는 것으로한다.

앞에서 논의한 바와 같이 랩 배포 시 ServiceMeshControlPlane에서 mtls 및 automtls를 비활성화했습니다. 네임스페이스 수준에서 모든 것을 제어하고 다음 단계에서 mTLS를 비활성화하기를 원하기 때문입니다.

[![0](https://rcarrata.com/images/automtls.png)](https://rcarrata.com/images/automtls.png)

yaml 클러스터의 ServiceMeshControlPlane에서 Mesh Control Plane의 보안 부분에 대한 특정 CR을 볼 수 있습니다.

```
oc get smcp -n istio-system basic-install -o jsonpath='{.spec.security}' | jq .
{
  "controlPlane": {
    "mtls": false
  },
  "dataPlane": {
    "automtls": false,
    "mtls": false
  }
}
```

참고: PeerAuthentication은 기본적으로 Istio 컨트롤 플레인 네임스페이스(이 경우에는 istio-system)에서 생성됩니다.

이를 통해 클러스터에서 생성된 PeerAuthentications 및 대상 규칙을 완전히 제어할 수 있지만 다른 한편으로는 ServiceMeshControlPlane 및 Service Mesh Operators에 의해 자동화되지 않으며 각각의 경우에 제공해야 합니다.

중요: 프로덕션(또는 비 데모/poc 시스템)에서는 dataPlane과 controlPlane 모두에서 mtls, automtls를 활성화하고 네임스페이스 수준에서 PeerAuthentication을 제어하는 것이 좋습니다. 이전 설정은 PoC/Demo용입니다!!

이제 Service Mesh 컨트롤 플레인이 설치되었으므로 재미있게 합시다!

## C. 허용 모드에서 mTLS 배포

### C.1. 특정 앱에 대한 Service Mesh mTLS 배포(네임스페이스 수준)

앞에서 설명한 것처럼 [Service Mesh 설명서에서](https://docs.openshift.com/container-platform/4.9/service_mesh/v2x/ossm-security.html#ossm-security-mtls_ossm-security) 지정 하는 대로 네임스페이스 수준에서 mtls를 활성화할 수 있습니다 .

네임스페이스 수준에서 mTLS를 활성화하는 데 필요한 Istio 개체를 배포해 보겠습니다. 이 블로그 게시물에서는 잘 알려진 bookinfo 앱을 사용할 것입니다.

- 블로그 게시물 중에 사용된 몇 가지 편리한 변수를 먼저 내보냅니다.

```
export bookinfo_namespace=bookinfo
export control_plane_namespace=istio-system
export control_plane_name=basic-install
export control_plane_route_name=api
```

- 그런 다음 Bookinfo 앱과 Helm v3를 통해 배포하는 mTLS를 활성화하는 데 필요한 모든 istio 개체를 설치합니다.

```
DEPLOY_NAMESPACE=${bookinfo_namespace}
CONTROL_PLANE_NAMESPACE=${control_plane_namespace}
CONTROL_PLANE_NAME=${control_plane_name}
CONTROL_PLANE_ROUTE_NAME=${control_plane_route_name}

oc project ${DEPLOY_NAMESPACE}

helm upgrade -i basic-gw-config -n ${DEPLOY_NAMESPACE} \
  --set control_plane_namespace=${CONTROL_PLANE_NAMESPACE} \
  --set control_plane_name=${CONTROL_PLANE_NAME} \
  --set route_hostname=$(oc get route ${CONTROL_PLANE_ROUTE_NAME} -n ${CONTROL_PLANE_NAMESPACE} -o jsonpath={'.spec.host'}) \
  basic-gw-config -f basic-gw-config/values-mtls.yaml
```

- Bookinfo 네임스페이스에서 생성된 [PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/) 개체를 확인합니다 .

```
oc get peerauthentications.security.istio.io default -o jsonpath='{.spec}' -n $bookinfo_namespace | jq .
{
  "mtls": {
    "mode": "PERMISSIVE"
  }
}
```

PeerAuthentication은 트래픽이 사이드카로 터널링되는지 여부를 정의합니다. PERMISSIVE 모드에서 연결은 일반 텍스트 또는 mTLS 터널일 수 있습니다.

자세한 내용 [은 PeerAuthentication 모드를](https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS-Mode) 확인하십시오 .

### C.2. 외부 사용자로 Bookinfo 테스트(OCP 클러스터 외부)

OCP 클러스터 외부에서 앱을 테스트하여 애플리케이션 사용자인 것처럼 시뮬레이션해 보겠습니다.

```
GATEWAY_URL=$(echo https://$(oc get route ${control_plane_route_name} -n ${control_plane_namespace} -o jsonpath={'.spec.host'})/productpage)

curl $GATEWAY_URL -Iv
```

결과는 아래 컬이 설명하는 것처럼 200 OK여야 합니다.

```
...
> HEAD /productpage HTTP/1.1
> Host: api-istio-system.apps.XXX.com
> User-Agent: curl/7.76.1
> Accept: */*
>
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
content-type: text/html; charset=utf-8
< content-length: 4183
content-length: 4183
< server: istio-envoy
server: istio-envoy
...
<
* Connection #0 to host api-istio-system.apps.XXXX.com left intact
```

### C.3. 키알리 서비스 확인

- Kiali 서비스를 확인합시다.

```
echo https://$(oc get route -n $control_plane_namespace kiali -o jsonpath={'.spec.host'})
```

- Kiali에서 볼 수 있습니다(mTLS 보안 플래그가 활성화된 상태). 요청에서 ingressGW를 생각하고 앱의 마이크로 서비스 간의 모든 트래픽이 mTLS로 암호화됨을 알 수 있습니다.

[![MTL 1](https://rcarrata.com/images/mtls-kiali1.png)](https://rcarrata.com/images/mtls-kiali1.png)

## C.4. 메시 내부/내부의 mTLS 확인

메쉬 응용 프로그램 내부에서 테스트를 수행하여 요청이 메쉬 내부가 아닌 다른 포드에서 오는 것을 시뮬레이션해 보겠습니다.

- 테스트를 실행하기 위해 테스트 애플리케이션을 배포합니다.

```
oc create deployment --image nginxinc/nginx-unprivileged inside-mesh -n $bookinfo_namespace
```

- 자동 삽입을 활성화하고 sidecar.istio.io/inject 주석을 true로 추가하여 애플리케이션 배포를 패치합니다.

```
oc patch deploy/inside-mesh -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n bookinfo
```

- 포드 내에서 ProductPage 마이크로 서비스에 대한 여러 요청을 실행합니다.

```
oc exec -ti deploy/inside-mesh -- bash
$ for i in {1..10}; do curl -vI http://productpage:9080/productpage?u=normal && sleep 2; done
```

- 모든 요청은 HTTP 200 OK를 반환해야 합니다.

[![MTL 2](https://rcarrata.com/images/mtls-kiali-inside-mesh.png)](https://rcarrata.com/images/mtls-kiali-inside-mesh.png)

눈치채셨다면 요청은 Ingress Gw에서 오는 것이 아니라 Inside-Mesh 포드에서 오는 것입니다. 멋지지 않나요?

### C.5. 메시 외부의 mTLS 확인

이제 PERMISSIVE 모드를 사용하여 Mesh 네임스페이스에서 허용되는 트래픽이 mTLS 및 일반 텍스트(HTTP)에 있을 수 있음을 보여줄 차례입니다.

- Istio 사이드카를 주입하지 않고 테스트를 실행하기 위해 다른 배포를 생성해 보겠습니다.

```
oc create deployment --image nginxinc/nginx-unprivileged outside-mesh -n $bookinfo_namespace
```

- 외부 포드에서 이전과 동일한 요청을 실행합니다.

```
oc exec -ti deploy/outside-mesh -- bash
$ for i in {1..10}; do  curl -vI http://productpage:9080/productpage?u=normal && sleep 2; done
```

참고: 모든 요청은 HTTP 200 OK를 반환해야 합니다.

[![mtls 3](https://rcarrata.com/images/mtls-kiali-outside-mesh.png)](https://rcarrata.com/images/mtls-kiali-outside-mesh.png)

"알 수 없음"에서 시작된 요청이 있음을 알 수 있습니다. 그것이 curl 명령이 실행된 내부의 포드입니다.

### C.6 허용 모드에서 mTLS를 사용하여 마이크로서비스 간 암호화 확인

이제 우리 트래픽이 [istioctl 도구를](https://istio.io/latest/docs/setup/getting-started/#download) 사용하여 암호화 [되었는지 확인](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-experimental-authz-check) 하고 [실험적 authz 검사](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-experimental-authz-check) 플래그를 포드 중 하나에서 직접 [확인](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-experimental-authz-check) 합니다(무작위로 세부 정보를 선택했지만 다른 것과 작동함).

- Pod에서 Envoy 구성의 상태를 확인하기 위해 Pod id 및 grepping을 사용하여 실험적 authz 검사를 실행해 보겠습니다.

```
istioctl experimental authz check $(kubectl get pods -n bookinfo | grep details| head -1| awk '{print $1}') | grep virtual
Checked 13/25 listeners with node IP 10.131.1.100.
LISTENER[FilterChain]     CERTIFICATE          mTLS (MODE)          AuthZ (RULES)
...
virtualOutbound[0]        none                 no (none)            no (none)
virtualOutbound[1]        none                 no (none)            no (none)
virtualInbound[0]         none                 no (none)            no (none)
virtualInbound[1]         noneSDS: default     yes (none)           no (none)
virtualInbound[2]         none                 no (none)            no (none)
virtualInbound[3]         noneSDS: default     yes (none)           no (none)
virtualInbound[4]         none                 no (none)            no (none)
virtualInbound[5]         none                 no (none)            no (none)
virtualInbound[6]         noneSDS: default     yes (PERMISSIVE)     no (none)
virtualInbound[7]         none                 no (PERMISSIVE)      no (none)
virtualInbound[8]         noneSDS: default     yes (PERMISSIVE)     no (none)
virtualInbound[9]         none                 no (PERMISSIVE)      no (none)
0.0.0.0_15010             none                 no (none)            no (none)
0.0.0.0_15014             none                 no (none)            no (none)
```

가상 인바운드에서 mTLS가 활성화되어 있습니다.

- [Istio](https://github.com/eldadru/ksniff) 프록시를 통해 트래픽이 전송되는 방식을 확인하기 위해 [Ksniff](https://github.com/eldadru/ksniff) 유틸리티를 사용할 것입니다.

```
POD_PRODUCTPAGE=$(oc get pod | grep productpage | awk '{print $1}')

DETAILS_IP=$(oc get pod -o wide | grep details | awk '{print $6}')
echo $DETAILS_IP
```

이전의 경우 세부 정보 IP는 10.131.1.100입니다.

- 그런 다음 다음을 실행하여 ProductPage 포드에서 Details v1 포드로 전송되는 트래픽을 스니핑해 보겠습니다.

```
kubectl sniff -i eth0 -o ./mtls.pcap $POD_PRODUCTPAGE -f '((tcp) and (net 10.131.1.100))' -n bookinfo -p -c istio-proxy
INFO[0000] sniffing method: privileged pod
INFO[0000] sniffing on pod: 'productpage-v1-658849bb5-lfrh9' [namespace: 'bookinfo', container: 'istio-proxy', filter: '((tcp) and (net 10.131.1.100))', interface: 'eth0']
INFO[0000] creating privileged pod on node: 'xxxx'
INFO[0000] pod: 'ksniff-wn8t4' created successfully in namespace: 'bookinfo'
INFO[0000] waiting for pod successful startup
INFO[0007] pod: 'ksniff-wn8t4' created successfully on node: 'xxxxx'
INFO[0007] executing command: '[chroot /host crictl inspect --output json ac2bda75954daf1c4e4ab48c0ffdf7a59a36cb83cbbb9649d89deb420f572ab4]' on container: 'ksniff-privileged', pod: 'ksniff-wn8t4', namespace: 'bookinfo'
INFO[0008] command: '[chroot /host crictl inspect --output json ac2bda75954daf1c4e4ab48c0ffdf7a59a36cb83cbbb9649d89deb420f572ab4]' executing successfully exitCode: '0', stdErr :''
INFO[0008] output file option specified, storing output in: './mtls.pcap'
INFO[0008] starting remote sniffing using privileged pod
INFO[0008] executing command: '[nsenter -n -t 2430077 -- tcpdump -i eth0 -U -w - ((tcp) and (net 10.131.1.100))]' on container: 'ksniff-privileged', pod: 'ksniff-wn8t4', namespace: 'bookinfo'
```

- 이제 새 터미널 창으로 이동하여 다음을 실행합니다.

```
curl $GATEWAY_URL -I
```

- 그런 다음 kubectl sniff 터미널 창으로 이동하여 프로세스를 중지합니다(Ctrl+C).
- 동일한 디렉토리에 캡처된 트래픽인 mtls.pcap이라는 파일이 있고 Wireshark를 사용하여 파일을 열 수 있으며 다음과 같은 내용을 볼 수 있습니다.

[![MTL 4](https://rcarrata.com/images/mtls-enabled-wireshark.png)](https://rcarrata.com/images/mtls-enabled-wireshark.png)

Wireshark로 pcap 파일을 확인하면 PERMISSIVE 모드로 PeerAuthentication 개체에서 mTLS 기능을 활성화했기 때문에 모든 마이크로서비스에서 들어오고 나가는 모든 트래픽이 TLS1.x로 암호화됩니다.

## D. DISABLE 모드로 mTLS 배포

이제 네임스페이스 수준에서 mTLS를 끄면 어떻게 되는지 살펴보겠습니다. PeerAuthentication 수준에서 DISABLE 모드를 사용하여 mTLS를 비활성화하고 DestinationRule 수준에서도 적용할 수 있습니다(강력한 요구 사항은 아니지만 모든 위치에서 DISABLE 모드가 설정되어 있음을 보장합니다).

- bookinfo 네임스페이스에서 mTLS를 비활성화하기 위해 Istio 개체를 배포해 보겠습니다.

```
helm upgrade -i basic-gw-config -n ${DEPLOY_NAMESPACE} \
  --set control_plane_namespace=${CONTROL_PLANE_NAMESPACE} \
  --set control_plane_name=${CONTROL_PLANE_NAME} \
  --set route_hostname=$(oc get route ${CONTROL_PLANE_ROUTE_NAME} -n ${CONTROL_PLANE_NAMESPACE} -o jsonpath={'.spec.host'}) \
  basic-gw-config -f basic-gw-config/values-mtls-disabled.yaml
```

- PeerAuthentication이 DISABLE 모드인지 확인하기 위해 확인하십시오.

```
oc get peerauthentications.security.istio.io -n bookinfo default -o jsonpath='{.spec}' | jq .
{
  "mtls": {
    "mode": "DISABLE"
  }
}
```

- DestinationRule을 확인하여 DISABLE 모드에 있는지 확인하십시오.

```
oc get dr {-n bookinfo bookinfo-mtls -o jsonpath='{.spec}' | jq .
  "host": "*",
  "trafficPolicy": {
    "tls": {
      "mode": "DISABLE"
    }
  }
}
```

- 이제 이것이 Service Mesh 클러스터의 Envoy Istio 포드에도 효과적인 경우 하나의 임의 포드를 체크인합니다.

```
istioctl experimental authz check $(kubectl get pods -n bookinfo | grep details| head -1| awk '{
print $1}')
Checked 13/25 listeners with node IP 10.131.1.100.
LISTENER[FilterChain]     CERTIFICATE     mTLS (MODE)     AuthZ (RULES)
0.0.0.0_80                none            no (none)       no (none)
0.0.0.0_3000              none            no (none)       no (none)
0.0.0.0_5778              none            no (none)       no (none)
0.0.0.0_9080              none            no (none)       no (none)
0.0.0.0_9090              none            no (none)       no (none)
0.0.0.0_9411              none            no (none)       no (none)
0.0.0.0_14250             none            no (none)       no (none)
0.0.0.0_14267             none            no (none)       no (none)
0.0.0.0_14268             none            no (none)       no (none)
virtualOutbound[0]        none            no (none)       no (none)
virtualOutbound[1]        none            no (none)       no (none)
virtualInbound[0]         none            no (none)       no (none)
virtualInbound[1]         none            no (none)       no (none)
virtualInbound[2]         none            no (none)       no (none)
virtualInbound[3]         none            no (none)       no (none)
virtualInbound[4]         none            no (none)       no (none)
virtualInbound[5]         none            no (none)       no (none)
0.0.0.0_15010             none            no (none)       no (none)
0.0.0.0_15014             none            no (none)       no (none)
```

가상 인바운드에서는 mTLS가 활성화되어 있지 않습니다.

트래픽이 Istio 프록시를 통해 일반 텍스트로 전송되는 방식을 확인하기 위해 [Ksniff](https://github.com/eldadru/ksniff) 유틸리티를 사용할 것입니다.

- ProductPage 포드 ID 및 세부 정보 IP를 추출합니다(ProductPage 포드에서 확인을 시작하고 세부 정보 포드 IP에 대한 트래픽을 확인합니다).

```
POD_PRODUCTPAGE=$(oc get pod | grep productpage | awk '{print $1}')

DETAILS_IP=$(oc get pod -o wide | grep details | awk '{print $6}')
echo $DETAILS_IP
```

이전의 경우 세부 정보 IP는 여전히 10.131.1.100입니다.

- 그런 다음 다음을 실행하여 ProductPage 포드에서 Details v1 포드로 전송되는 트래픽을 스니핑해 보겠습니다.

```
kubectl sniff -i eth0 -o ./mtls-disabled.pcap $POD_PRODUCTPAGE -f '((tcp) and (net 10.131.1.100))' -n bookinfo -p -c istio-proxy
```

- Kniff가 실행되면 몇 가지 요청을 수행해야 합니다.

```
curl $GATEWAY_URL -I
```

- 그런 다음 kubectl sniff 터미널 창으로 이동하여 프로세스를 중지합니다(Ctrl+C).
- 동일한 디렉토리에 캡처된 트래픽인 mtls-disabled.pcap이라는 파일이 있고 Wireshark를 사용하여 파일을 열 수 있으며 다음과 같은 내용을 볼 수 있습니다.

[![MTL 5](https://rcarrata.com/images/mtls-disabled-wireshark.png)](https://rcarrata.com/images/mtls-disabled-wireshark.png)

Wireshark로 pcap 파일을 확인하면 이제 트래픽이 암호화되지 않고 일반 텍스트로 표시됩니다! 따라서 누군가 중간자 공격을 하면 민감한 정보가 누출될 수 있습니다!

중요: 이것은 프로덕션에 권장되지 않지만 이미 알고 있지 않습니까? :디

## E. STRICT로 mTLS 시행

마지막으로 mTLS 모드 STRICT를 사용하여 모든 인바운드 및 아웃바운드 트래픽에서 mTLS 사용을 적용해 보겠습니다.

## 마.1. STRICT mTLS 값 활성화(PeerAuth 제외)

- 이번에는 mTLS를 STRICT로 활성화하는 설정을 배포해 보겠습니다.

```
helm upgrade -i basic-gw-config -n ${DEPLOY_NAMESPACE} \
  --set control_plane_namespace=${CONTROL_PLANE_NAMESPACE} \
  --set control_plane_name=${CONTROL_PLANE_NAME} \
  --set route_hostname=$(oc get route ${CONTROL_PLANE_ROUTE_NAME} -n ${CONTROL_PLANE_NAMESPACE} -o jsonpath={'.spec.host'}) \
  basic-gw-config -f basic-gw-config/values-mtls-strict.yaml
```

- STRICT 모드가 설정되어 있는지 PeerAuthentication을 확인하십시오.

```
oc get peerauthentications.security.istio.io default -o jsonpath='{.spec}' | jq .
{
  "mtls": {
    "mode": "STRICT"
  }
}
```

STRICT 모드에서 연결은 mTLS 터널입니다(클라이언트 인증서가 있는 TLS가 제공되어야 함).

참고: 메시가 PeerAuthentication 개체를 적용하기 시작하려면 시간이 걸릴 수 있습니다. 몇 분 정도 기다려 주십시오.

반면에 자동 mTLS를 활성화하지 않았고 STRICT에 대한 PeerAuthentication이 있으므로 애플리케이션 서비스에 대한 DestinationRule 리소스를 생성해야 합니다.

- 메시의 다른 서비스에 요청을 보낼 때 mTLS를 사용하도록 Maistra를 구성하는 대상 규칙을 만듭니다.

```
oc get dr bookinfo-mtls -o jsonpath='{.spec}' | jq .
{
  "host": "*.bookinfo.svc.cluster.local",
  "trafficPolicy": {
    "tls": {
      "mode": "ISTIO_MUTUAL"
    }
  }
}
```

### E.2. 외부 사용자로 Bookinfo 테스트(OCP 클러스터 외부)

STRICT 모드가 적용된 상태에서 모든 것이 예상대로 작동했는지 확인합시다!

- 우리가 일반 사용자라고 시뮬레이션하여 클러스터 외부에서 몇 가지 요청을 수행해 보겠습니다.

```
GATEWAY_URL=$(echo https://$(oc get route ${control_plane_route_name} -n ${control_plane_namespace} -o jsonpath={'.spec.host'})/productpage)

curl $GATEWAY_URL -I
```

- 예상대로 요청은 모두 200 OK이므로 사랑하는 사용자를 위해 모든 것이 좋습니다!

```
$ curl $GATEWAY_URL -I
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 5179
server: istio-envoy
date: Tue, 26 Oct 2021 14:59:54 GMT
x-envoy-upstream-service-time: 54
set-cookie: 9023cfc9bfd45f5dc28756be7ac3bd3e=d8c5201aae101ce45ff167c09f1d6910; path=/; HttpOnly; Secure; SameSite=None
cache-control: private
```

### E.3 메시 내부/내부의 mTLS 확인

메쉬 내부와 OpenShift 클러스터 내부의 포드 내부를 확인해 보겠습니다.

- 포드 내부에서 몇 가지 요청을 실행해 보겠습니다.

```
oc exec -ti deploy/inside-mesh -- bash
$ for i in {1..10}; do  curl -vI http://productpage:9080/productpage?u=normal && sleep 2; done
```

- 우리가 볼 수 있듯이 모든 요청은 HTTP 200 OK를 반환해야 합니다.

[![MTL 5](https://rcarrata.com/images/mtls-kiali-inside-mesh.png)](https://rcarrata.com/images/mtls-kiali-inside-mesh.png)

### E.4. 메시 외부의 mTLS 확인

- 메시 외부(그러나 OpenShift 클러스터 내부)에서 연결을 다시 테스트해 보겠습니다.

```
oc exec -n bookinfo -ti deployment/outside-mesh -- bash
```

- 허용 모드를 설정할 때와 완전히 동일한 일부 요청을 실행해 보겠습니다.

```
1000760000@outside-mesh-847dd9c646-6vj2b:/$ curl http://productpage:9080/productpage?u=normal -Iv
...
...
* Expire in 200 ms for 4 (transfer 0x560642d40c10)
* Connected to productpage (172.30.64.160) port 9080 (#0)
> HEAD /productpage?u=normal HTTP/1.1
> Host: productpage:9080
> User-Agent: curl/7.64.0
> Accept: */*
>
* Recv failure: Connection reset by peer
* Closing connection 0
curl: (56) Recv failure: Connection reset by peer
```

컬 파드의 컬이 종료 코드 56과 함께 실패하고 실패!! 무슨 일이야?

이는 이제 기본 설정에서 피어 인증 Istio 개체를 통한 상호 TLS(STRICT)를 통한 암호화된 통신이 필요하지만 컬 포드(메시 외부에 있음)가 상호 TLS를 사용하려고 시도하지 않기 때문입니다.

STRICT 모드를 사용하면 워크로드/마이크로서비스가 암호화된 연결(mTLS)만 허용하도록 강제할 수 있지만 메시의 서비스가 메시 외부의 서비스와 통신하는 경우 엄격한 mTLS가 해당 서비스 간의 통신을 중단할 수 있으므로 주의해야 합니다. 워크로드를 OpenShift Service Mesh로 마이그레이션하는 동안 허용 모드를 사용하십시오!

이것으로 Service Mesh의 mTLS에 대한 심층 분석을 마치겠습니다!

Asier Cidón, Fran Perea 및 Ernesto Gonzalez의 지혜와 인내심, 그리고 항상 기꺼이 도와주신 것에 대해 특별히 감사드립니다! 당신은 당신을 바위!

누구에게나 도움이 되길 바라며 해피메싱!!
