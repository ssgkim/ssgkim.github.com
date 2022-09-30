---
title: 오픈시프트 TCP 라우터
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
description: 라우터 활용하기
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-07-15 23:00:00 +0800'
---

# 오픈시프트 TCP 라우터

이 문서에서는 haproxy 템플릿을 사용자 지정하고 경로에 주석을 추가하여 외부 세계로 tcp 경로를 여는 방법을 설명합니다. 이 문서와 관련된 스크립트 및 파일은 scripts 디렉토리에 있습니다.

## 본래 의도와 원칙

우리는 openshift의 poc에서 L4 로드 밸런싱 테스트를 자주 접합니다. 우리는 기본 ocp 라우터가 haproxy에 의해 만들어지고 기본적으로 http와 https만 지원한다는 것을 알고 있습니다. tls/sni도 tcp를 지원하는 방법이지만 이것은 여전히 7개의 레이어입니다. 공식 문서에는 다른 요구 사항이 있는 경우 이를 충족하도록 haproxy 템플릿을 사용자 정의할 수 있지만 사용자 정의가 거의 없고 예제가 많지 않다고 간단히 나와 있습니다. 이 문서는 경로 구성을 동적으로 모니터링하고 tcp 포트를 동적으로 여는 사용자 지정 haproxy 템플릿입니다.

haproxy 템플릿을 사용자 정의하려면 openshift 라우터의 몇 가지 핵심 사항을 이해해야 합니다.

- openshift 라우터는 haproxy일 뿐만 아니라 openshift의 구성을 모니터링하고 haproxy 템플릿을 구성하기 위한 매우 중요한 구성 파일인 많은 맵 파일을 작성하는 go 프로그램을 가지고 있습니다.
- 오픈시프트 라우터의 tls 패스스루 방식은 tcp 모드인 haproxy 설정에 해당하며, 우리의 커스터마이징 포인트는 여기입니다.
- 사용자 지정 프로세스는 http/https의 에지를 보호하고 부분을 다시 암호화하고 주석 경로에 대한 tls 패스스루의 프런트엔드를 여는 데 중점을 둡니다.
- 경로 주석 구성 양식은 haproxy.router.openshift.io/external-tcp-port: "13306"입니다.
- 물론 ocp4는 아직 사용자 지정 경로 템플릿을 지원하지 않으므로 이 기사에서는 경로 배포를 직접 생성합니다.
- 현장 구현 시 공유기 이미지 변경에 주의 하시고, 각 버전별 이미지는 release.txt 파일에서 확인하실 수 있습니다.



다음 방안은 POC를 위한 편향적인 방안으로 한계를 가지고 있음

- 라우트 어노테이션에 의해 정의된 개방형 tcp 포트는 수동으로 정의되고 전체 클러스터의 각 프로젝트에 열려 있으므로 필연적으로 tcp 포트 충돌이 발생합니다. 

다음은 route의 구성 예입니다.

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: ottcache-002
  annotations:
    haproxy.router.openshift.io/wzh-router-name: "wzh-router-1"
    haproxy.router.openshift.io/external-tcp-port: "6620"
spec:
  to:
    kind: Service
    name: ottcache-002-service
  port:
    targetPort: 6620
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: None
```

다음은 템플릿의 주요 사용자 지정 지점입니다.

```go
{{/*try to add tcp support*/}}

{{- if eq (env "WZH_ROUTER_NAME" "wzh-router-name") (index $cfg.Annotations "haproxy.router.openshift.io/wzh-router-name") }}
  {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/external-tcp-port")) }} 
  frontend tcp-{{ (index $cfg.Annotations "haproxy.router.openshift.io/external-tcp-port") }}
    bind *:{{ (index $cfg.Annotations "haproxy.router.openshift.io/external-tcp-port") }}
    mode tcp
    default_backend {{genBackendNamePrefix $cfg.TLSTermination}}:{{$cfgIdx}}

  {{- end}}{{/* end haproxy.router.openshift.io */}}
{{- end}}{{/* end WZH_ROUTER_NAME */}}

{{/*end try to add tcp support*/}}
```

## [테스트 단계]

테스트 단계는 복잡하지 않으며 새 라우터를 생성한 다음 다른 프로젝트로 이동하여 애플리케이션을 생성하고 경로에 주석을 달기만 하면 됩니다.

이 기사의 예제에는 두 개의 애플리케이션이 포함되어 있습니다. 하나는 웹 애플리케이션이고 다른 하나는 mysql입니다. 둘 다 tcp 포트를 통해 외부 세계에 열려 있습니다.

```bash
# tcp-router will install in the same project with openshift router
oc project openshift-ingress

# install the tcp-router and demo
oc create configmap customrouter-wzh --from-file=haproxy-config.template
oc apply -f haproxy.router.yaml

oc apply -f haproxy.demo.yaml

# test your tcp-router, replace ip with router ip, both command will success.
curl 192.168.7.18:18080

podman run -it --rm registry.redhat.ren:5443/docker.io/mysql mysql -h 192.168.7.18 -P 13306 -u user -D db -p

# if you want to delete the tcp-router and demo
oc delete -f haproxy.router.yaml
oc delete configmap customrouter-wzh

oc delete -f haproxy.demo.yaml

# oc set volume deployment/router-wzh --add --overwrite \
#     --name=config-volume \
#     --mount-path=/var/lib/haproxy/conf/custom \
#     --source='{"configMap": { "name": "customrouter-wzh"}}'

# oc set env dc/router \
#     TEMPLATE_FILE=/var/lib/haproxy/conf/custom/haproxy-config.template
```

## Reference

https://docs.openshift.com/container-platform/3.11/install_config/router/customized_haproxy_router.html#go-template-actions

https://www.haproxy.com/blog/introduction-to-haproxy-maps/

https://access.redhat.com/solutions/3495011

https://blog.zhaw.ch/icclab/openshift-custom-router-with-tcpsni-support/

## 기타 방법

소스코드를 분석해보면 오픈시프트 라우터가 haproxy를 확장한 것을 볼 수 있는데, 그 맵 파일은 모두 라우터 확장에 의해 생성되며, 목적은 엔드포인트를 연결하고 서비스를 우회하는 것입니다. 그래서 우리는 tcp 포워딩을 하고 싶고, sni-tcp를 사용하여 tcp 포워딩을 할 수 있습니다. ![img](https://wangzheng422.github.io/docker_env/ocp4/4.3/imgs/2020-02-23-14-04-49.png)


{: .notice--info}

**참고자료** <br>
-- [https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators]({{"https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators"}}){:target="_blank"}<br>
{: .notice--info}
