---
title: OpenTelemetry 추적 수집 및 시각화
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
description: OpenTelemetry는 추적, 메트릭 및 로그와 같은 원격 분석 데이터를 처리하기 위한 API 및 도구를 제공하는 관찰 가능성 프레임워크입니다.
article_tag1: OpenShift
article_tag2: OpenTelemerty
article_tag3: WAS
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-03-26 23:00:00 +0800'
---


### OpenTelemetry 및 OTLP(OpenTelemetry Protocol)에 대한 내용도 관심있게 봐야 할 내용입니다.

우리는 추적 데이터를 생성하고 내보내기 위해 Red Hat OpenShift 및 Kubernetes용 컨테이너 엔진인 CRI-O를 계측했습니다. 여기서 중요한 점은 OpenTelemetry 추적의 수집 및 시각화 방법입니다. Kubernetes와 OpenShift에서 CRI-O, APIServer 및 Etcd 추적을 확인하는 방법이 무엇일까요?

Kubernetes와 같은 복잡한 시스템에서 분산 추적은 클러스터 성능 문제를 해결하는 데 걸리는 시간을 줄입니다. 추적은 시스템을 통해 이동할 때 트랜잭션에 대한 명확한 보기를 제공합니다. 데이터는 종단 간 요청을 구성하는 여러 프로세스 및 서비스에서 상호 연관됩니다.

추적이 제공하는 컨텍스트 전파와 OTLP 데이터의 일관된 구조는 메트릭에서만 수집된 정보를 향상시킵니다. Kubernetes 및 클라우드 네이티브 애플리케이션이 더욱 분산되고 복잡해짐에 따라 추적은 서비스와 워크로드를 이해하고 디버그하는 데 필수적입니다.

OpenTelemetry Collector를 사용하여 OTLP 추적을 수집하고 추적 데이터를 시각화할 수 있는 오픈 소스 소프트웨어인 Jaeger 로 내보내는 방법을 알아봅니다. CRI-O, APIServer 및 Etcd에서 수집 추적을 캡처하고 다른 하나는 다중 노드 OpenShift 클러스터에서 수집된 CRI-O 추적을 확인합니다. 또한 스팬을 수집하고 내보내는 데 필요한 구성에 대한 개요도 살펴봅니다. 

#### CRI-O, APIServer 및 etcd 추적이 있는 Kubernetes
최신 Kubernetes APIServer 릴리스(1.22) 및 최신 버전의 etcd(3.5.0)에는 실험적 OpenTelemetry 추적 내보내기가 포함됩니다.

#### 추적 수집 개요
각 CRI-O 서버의 추적 내보내기는 각 노드의 0.0.0.0:4317에서 에이전트 포드에 연결하여 OTLP 데이터를 내보냅니다. 호스트에서 OTLP를 수신하면 각 에이전트 포드는 OTLP 데이터를 에이전트와 동일한 네임스페이스에서 실행되는 단일 OpenTelemetry Collector 배포 및 포드로 내보냅니다. OpenTelemetry Collector에서 OTLP 데이터는 선택한 백엔드(이 경우 Jaeger)로 내보내집니다. 

#### CRI-O 추적 수집에는 다음 단계가 포함됩니다.

- OpenTelemetry-Agent DaemonSet 및 OpenTelemetry Collector 배포가 클러스터에 설치됩니다.
- 에이전트 포드는 CRI-O, APIServer 및 Etcd에서 OTLP 데이터를 수신합니다. 그런 다음 에이전트는 OTLP 데이터를 OpenTelemetry Collector로 내보냅니다.
- Jaeger Operator가 설치되어 Jaeger Custom Resources를 감시합니다.
- Jaeger Custom Resource는 OpenTelemetry Collector와 동일한 네임스페이스에 생성됩니다.
- OpenTelemetry Collector 팟(Pod)은 에이전트에서 OTLP 데이터를 수신하고 OTLP 데이터를 Jaeger 팟(Pod)으로 내보냅니다.
- 추적 데이터는 Jaeger 프런트엔드와 함께 표시됩니다.



{: .notice--info}

**참고자료** <br>
-- [https://next.redhat.com/2021/11/04/collecting-and-visualizing-opentelemetry-traces/]({{"https://next.redhat.com/2021/11/04/collecting-and-visualizing-opentelemetry-traces/"}}){:target="_blank"} <br>
{: .notice--info}
