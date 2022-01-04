---
title: OpenShift EFK 스택 멀티라인 로그
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
description: Java Stack trace와 같은 여러줄 로그를 EFK에서 효율적으로 확인하는 방법을 알아봅니다.
article_tag1: OpenShift
article_tag2: logging
article_tag3: infra
article_section: IT근황 공유하기
meta_keywords: openshift,kubernetes,cncf,infra,network
last_modified_at: '2021-12-04 23:00:00 +0800'
---

컨테이너에서 애플리케이션을 실행하면 수평적 확장 및 효과적인 리소스 관리와 같은 많은 이점이 있습니다. [클라우드 네이티브](https://www.cncf.io/) 애플리케이션을 개발 하려면 다른 사고방식이 필요하며 새로운 프레임워크에 익숙해지는 데에는 항상 학습 곡선이 있습니다. 배포 관리, 구성, 메트릭, 보안, 로깅, 모니터링, [서비스 조회와 같이](https://developers.redhat.com/books/introducing-istio-service-mesh-microservices/) 애플리케이션 서버에서 전통적으로 제공했던 많은 기능이 이제 컨테이너 런타임 환경에 속합니다. 컨테이너가 특정 언어가 아니며 애플리케이션이 핵심 비즈니스 로직에 집중할 수 있기 때문에 이는 매우 좋습니다. 반면에 클라우드 네이티브 앱은 본질적으로 동적이며 분산되어 로깅 및 추적과 같은 특정 작업을 더 복잡하게 만듭니다.

[Kubernetes](https://kubernetes.io/) 는 현재 가장 잘 알려진 컨테이너 런타임일 것입니다. [Red Hat OpenShift](https://www.openshift.com/) 는 Kubernetes를 기반으로 하는 엔터프라이즈 지원 컨테이너 애플리케이션 플랫폼입니다. 컨테이너에서 로그를 수집하고 검색하는 문제를 해결하려면 EFK(Elasticsearch, Fluentd, Kibana) 스택을 사용하여 [로그 집계](https://docs.openshift.com/container-platform/3.9/install_config/aggregate_logging.html) 를 [배포할](https://docs.openshift.com/container-platform/3.9/install_config/aggregate_logging.html) 수 있습니다 . 전체 로깅 솔루션을 이해하고 조정하는 것은 복잡할 수 있고 각 구성 요소에 대한 심층적인 검토가 필요하지만 일반적인 목적은 컨테이너의 표준 출력에서 로그 라인을 수집 및 인덱싱하고 Kubernetes 메타데이터로 레이블을 지정하는 것입니다. Docker 로그는 각 노드의 Fluentd 프로세스에 의해 수집되고 Elasticsearch로 전달되어 저장되며 Kibana는 조회를 위한 UI를 제공합니다.

![img](https://miro.medium.com/max/1400/1*_LD8Q4OlYGoZa2G1MFJYpA.png)

클러스터에서 첫 번째 컨테이너를 부팅한 후 알아차릴 수 있는 한 가지 문제는 표준 출력의 각 행이 별도의 로그 이벤트로 처리된다는 것입니다. 고유한 로그 행을 작성하는 한 괜찮지만 결국 애플리케이션에서 여러 행을 삭제하게 되는 특정 시나리오(예: 요청 페이로드, Java 스택 추적)가 있습니다. 줄 단위로 나누어져 있으면 찾아보거나 검색하는 것이 번거로우니 해결 방법을 찾아보도록 합시다.

문제는 Docker 수준에서 시작됩니다. Docker 프로세스의 표준 출력이 [journald](https://docs.docker.com/config/containers/logging/journald/) 또는 [json](https://docs.docker.com/config/containers/logging/json-file/) 로그로 끝나는지 여부에 관계없이 라인별로 처리됩니다. 이 수준에서 여러 줄을 그룹화하기 위한 기능 요청은 [막다른 골목](https://github.com/moby/moby/issues/22920#issuecomment-264036710) 이었습니다 . [fluentd](https://github.com/fluent-plugins-nursery/fluent-plugin-concat) 의 정규 표현식으로 여러 줄을 수집하는 것이 가능할 수도 있지만 현재로서는 일반적인 솔루션이 없는 것 같습니다. 언젠가 커뮤니티에서 받아들여지는 접근 방식을 제시하기를 희망하지만 주제에 대한 대부분의 토론은 "응용 프로그램에서 이 문제를 처리해야 합니다"라는 결론을 내립니다.

여기에서 애플리케이션에 약간의 영향을 주는 문제를 다소간 처리하는 두 가지 해결 방법을 시도합니다.

- 줄 끝을 CR '\r'로 변환
- 구조화된 json 로그 사용

> 이 게시물을 작성하는 동안 사용된 소프트웨어 버전:
> \- OpenShift v4.9
> \- Kubernetes v1.19
> \- OpenShift Logging 5.3
> \- Spring Boot v1.5.13
> \- Logback v1.1.11
> \- logstash-logback-encoder v5.1

# 라인 엔딩 해킹

전통적으로 다른 운영 체제는 다른 [줄 바꿈 문자를](https://en.wikipedia.org/wiki/Newline) 사용 [합니다](https://en.wikipedia.org/wiki/Newline) . 요즘 가장 많이 쓰이는 것은 Unix 스타일의 LF *(\n)* 와 Windows의 CRLF *(\r\n)* 이지만 과거에는 CR *(\r)* 도 자주 사용되었습니다. 최신 소프트웨어 스택은 서로 다른 스타일을 자동으로 인식하고 변환하여 다양한 줄 끝을 처리하는 수고를 덜어줍니다.

Openshift EFK 로깅 스택에서 놀랍게도 동일한 로그 이벤트에 속하는 줄에 대해 줄 바꿈으로 CR을 사용하고 로그 이벤트 사이에 일반 LF를 사용하는 것은 간단한 해킹으로 작동합니다. 컨테이너에 대해 이야기할 때 LF *(\n)* 줄 끝이 있는 Linux 환경에 대해 이야기 합니다. 따라서 어떻게든 애플리케이션의 로그 출력을 수정하여 하나의 로그 메시지 내의 모든 LF 문자를 CR로 변환해야 합니다. 이는 사용 중인 소프트웨어 스택에 따라 어려울 수도 있고 그렇지 않을 수도 있습니다.

저는 주로 Java 마이크로 서비스를 빌드하므로 Logback을 사용하여 Spring Boot 앱을 시도했습니다(Log4j에서도 동일하게 작동해야 함). 이 로깅 프레임워크에는 정규식을 문자열로 바꿀 수 있는 [*바꾸기*](https://logback.qos.ch/manual/layouts.html#replace) 기능이 있습니다. 이러한 로그 패턴은 다음 과 같이 *application.yml에* 정의할 수 있습니다 .

```
로깅: 
  패턴: 
    콘솔: %d %5p %logger: %replace(%m%wEx){'\\n','\r'}%nopex%n
```

기본적으로 LF(\n)를 CR(\r)로 대체하지만 표현에 약간의 설명이 필요하다. *대체* 변환은 메시지에 적용되는 `%m`하나있을 경우 자바 예외 스택 추적. `%wEx`인 [봄 부팅 특정](https://docs.spring.io/spring-boot/docs/1.5.13.RELEASE/api/org/springframework/boot/logging/logback/WhitespaceThrowableProxyConverter.html) 하지만, 같은 일부 공백의 주위로 예외입니다 `%n%ex`. 여기에 Java 문자열 이스케이프가 적용되어 첫 번째 매개변수는 2개의 문자 '\'+'n'의 정규 표현식이고 두 번째 매개변수는 하나의 CR 문자가 있는 문자열입니다( 아래 *logback.xml* 참조). 는 `%nopex`달리 Logback은 기본적으로 다시 추가, 우리는 패턴에서 예외 돌볼 것을 나타냅니다.

CR에 대한 유니코드 값을 사용하여 Logback 구성 XML에서 유사한 패턴을 설정할 수 있습니다.

참고: 패턴 레이아웃의 특수 CR 문자 때문에 OpenShift ConfigMap 또는 컨테이너 환경 변수(b64로 인코딩될 수 있음)를 통해 설정하는 것이 쉽지 않습니다. 일반적으로 로그 패턴은 애플리케이션과 함께 패키지된 속성 파일에 설정되므로 실제로 문제가 되지는 않습니다.

[이 예제 앱](https://github.com/bszeti/multiline-log) 은 줄 바꿈이 있는 메시지를 기록하고(줄 31) 시작할 때 예외를 throw합니다(줄 32). 기본적으로 모든 행은 별도의 로그 이벤트로 처리되었습니다. 위의 로그 패턴을 사용하면 OpenShift 콘솔의 포드 페이지에서 로그를 읽을 수 있습니다(로그 이벤트는 2K의 긴 세그먼트로 분할됨).

![img](https://miro.medium.com/max/2000/1*ktphH_HZLNJH2hZQZFodow.png)

또한 Kibana에서 로그 라인은 예상대로 하나의 로그 이벤트에 속합니다.

![img](https://miro.medium.com/max/2000/1*n3CL-GKjNEot1lwzn5RvSg.png)

로그 이벤트의 json 보기에는 '\r'로 인코딩된 줄 바꿈이 표시되지만 화면의 메시지 필드에서는 줄 바꿈이 잘 보입니다.

CR 문자는 OpenShift `oc`클라이언트로 포드 로그에 액세스하려고 하면 문제가 발생 하지만 간단한 스크립트(perl 또는 다른 것)를 사용하여 CR을 다시 LF로 변환할 수 있습니다.

```
oc 로그 -f multiline-log-9-sflkm | 펄 -pe 's/\R/\n/g'
```

# 구조화된 json 로그

위에서 설명한 줄 바꿈 문자 변환이 실제로 도움이 되지만 확실히 우연히 작동하는 것처럼 느껴집니다. "적절한 솔루션"으로 구조화된 로그를 사용해야 합니다. 이 경우 애플리케이션은 json 메시지의 각 로그 이벤트를 특정 필드 이름으로 래핑해야 하며, 여기서 개행은 '\n'으로 json으로 인코딩됩니다.

예를 들어, 두 줄의 초기화 로그 이벤트는 표준 출력에서 다음과 같습니다.

```
{" @timestamp ":"2018-07-30T17:15:01.127-04:00", " @version ":"1", "message":" 메시지가 있는 초기화 GreetingController:\napplication.yaml의 %s님 안녕하세요 ! ", "logger_name":"my.company.multilinelog.service.GreetingController", "thread_name":"main","level":"INFO","level_value":20000}
```

OpenShift [Fluentd 이미지](https://github.com/openshift/origin-aggregated-logging/tree/v3.9.0/fluentd) 는 이러한 json 로그를 구문 분석하고 이를 Elasticsearch로 전달된 메시지에 병합하는 사전 구성된 플러그인과 함께 제공됩니다. 이는 여러 줄 로그에 유용할 뿐만 아니라 로그 이벤트의 다른 필드(예: 타임스탬프, 심각도)가 Elasticsearch에서 보기 좋게 표시되도록 보장합니다.

Logback 또는 Log4j가 있는 Java 애플리케이션의 경우 [이 자세한 게시물을](https://developers.redhat.com/blog/2018/01/22/openshift-structured-application-logs/) 살펴보세요 . 우리는 추가 할 필요가 구성 로그를 사용하려면 [*logstash-logback-인코더*](https://github.com/logstash/logstash-logback-encoder) (참조 클래스 패스에 [pom.xml 파일을](https://github.com/bszeti/multiline-log/blob/master/pom.xml) ) 및 사용 *LogstashEncoder을* 이 같이 [*logback.xml*](https://github.com/bszeti/multiline-log/blob/master/src/main/resources/logback-logstash.xml) :

이 구성을 사용하면 Kibana에서 로그가 멋지게 보이지만 OpenShift 웹 콘솔에서 읽거나 `oc`json 메시지인 클라이언트를 사용하여 읽기가 어렵습니다 .

다른 개발 스택 및 언어에서 구조화된 로그를 활성화하는 방법을 알고 있다면 댓글에 추가하거나 이에 대한 게시물을 작성하세요…

# 요약

"여러 줄 로그를 처리할 수 있다는 것"이 단순한 요청처럼 들린다는 사실에도 불구하고 현재 사용 가능한 도구 집합으로 일반 솔루션을 찾는 것은 복잡합니다.

구조화된 로깅은 아마도 올바른 접근 방식일 수 있지만 모든 소프트웨어 개발 스택이 지원되는 구조에 동의하고 표준 출력에 "텍스트를 덤프"하는 것에서 멀어지는 것을 상상하기 어렵습니다. 이 패턴을 지원하는 프레임워크로 작업하더라도 프로덕션 환경에서만 유용합니다.

개발 또는 테스트 환경에서는 검색 기능보다 포드 표준 출력의 가독성이 더 중요할 수 있습니다. "라인 종료 해킹"은 중간 솔루션이며, 구조화된 json 로그에 대한 즉시 사용 가능한 솔루션이 없는 경우에도 구현하기가 더 쉬울 수 있습니다.
{: .notice--info}

**참고자료** <br>
-- [https://docs.openshift.com]({{"https://docs.openshift.com/container-platform/4.9/logging/cluster-logging-release-notes.html"}}){:target="_blank"} <br>
{: .notice--info}
