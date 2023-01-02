---
title: AI, 보안 및 에지의 미래
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
description: AI와 에지 그리고 그들의 미래
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-11-25 22:00:00 +0800'
---



# AI, 보안 및 에지의 미래

AI와 보안 그리고 에지에 대한 아주 재미있는 프로젝트가 생성되고 프로젝트들이 운영되고 있습니다. 이 중 레드햇에서 공유한 IoT 엣지 프로젝트를 공유할까 합니다.



최근 몇 년 동안 "에지 장치"는 단순한 IoT 센서에서 강력한 인공 지능(AI) 소프트웨어로 구동되는 자율 드론으로 진화했습니다. 마찬가지로 AI 소프트웨어를 개발하고 "엣지"에 배포하는 프로세스도 급속도로 발전했습니다. 오늘날 데이터 과학자는 애플리케이션의 중심에 있는 AI/ML 모델을 전문으로 하는 반면, 소프트웨어 공급망은 하이브리드 클라우드에서 에지까지 업데이트 및 데이터를 패키징하고 배송하는 일을 담당합니다.

최신 에지 장치에 대한 AI/ML 애플리케이션의 개발 및 배포를 위한 "커밋에서 프로덕션까지" 엔드 투 엔드 솔루션을 제시합니다. 당사의 솔루션은 새로운 오픈 소스 소프트웨어(OSS) 프로젝트를 기존 Red Hat 개방형 하이브리드 클라우드 도구 및 플랫폼과 통합합니다.



이 통합 프로젝트의 목표는 다음과 같습니다. 

첫째, AI/ML 모델을 개발하는 데이터 과학자의 속도를 높입니다. 

둘째, 자동화된 장치 증명 및 페이로드 서명 확인을 통해 소프트웨어 공급망의 보안을 강화합니다. 

셋째, 에지 장치를 Red Hat의 하이브리드 클라우드 플랫폼에 원활하게 통합하는 것을 보여줍니다.



통합 데모를 위해 자율 자율 주행 장난감 자동차용 OSS 툴킷인 [DonkeyCar 를 예제 AI 애플리케이션으로 사용합니다. ](https://www.donkeycar.com/)DonkeyCar 소프트웨어 제품군은 자율 주행 모델을 구축하고 교육하는 도구를 제공합니다. 이 데모에서는 Fedora 35가 설치된 Raspberry Pi 4에서 DonkeyCar 애플리케이션을 실행합니다. 자동차 애플리케이션 및 CI/CD 스크립트의 소스 저장소는 [여기에서](https://github.com/AICoE/integration-demo-summit-2022) 사용할 수 있습니다 .



## 배경

2022년에는 "엣지"의 새로운 정의인 AI 디바이스 에지가 등장했습니다. 예를 들어, 향후 몇 년 동안 널리 보급될 드론 기반 [음식 배달 서비스 를 생각해 보십시오. ](https://www.forbes.com/sites/saibala/2022/04/24/walgreens--alphabet-launch-inaugural-drone-deliveries-in-dfw/?sh=60ca96727c04)이 모델에서 자율 드론은 햄버거와 감자튀김과 같이 시간에 민감한 패키지를 임의의 주소로 배달하기 위해 주요 대도시 지역의 하늘을 탐색할 것입니다. 내부적으로는 고급 AI/ML 추론 소프트웨어가 GPU가 장착된 강력한 에지 장치 위에 배포됩니다.

이러한 새로운 현실을 보완하기 위해 Red Hat Emerging Technology 팀은 고객이 AI/ML 소프트웨어를 신속하게 개발하고 OpenShift 컨트롤 플레인을 통해 에지 장치에 안전하게 배포할 수 있도록 지원하는 도구를 개발했습니다. 이 게시물의 나머지 부분에서는 솔루션을 세 부분으로 나누어 간략하게 설명합니다.



## 1. 클라우드 네이티브 AI/ML 개발

데이터 과학자와 기계 학습 엔지니어는 오늘날 기술에서 [가장 수요가 많은 분야 중 하나입니다. ](https://www.linkedin.com/pulse/linkedin-jobs-rise-2022-25-us-roles-growing-demand-linkedin-news/)[이러한 역할은 일반적으로 Julia, Python 및 R과 같은 언어를 지원하는 JupyterHub](https://jupyter.org/) 와 같은 대화형 노트북에서 훈련된 AI/ML 모델을 생성하는 데 중점을 둡니다. 관리형 클라우드 서비스를 통해 이러한 노트북 환경을 클라우드에서 실행하고 액세스할 수 있습니다. 브라우저를 통해 개발자의 오버헤드를 크게 단순화합니다.

[Red Hat Emerging Technologies의 Project Meteor](https://operate-first-in-action.catalog.meteor.zone/README.html) 는 Red Hat 하이브리드 클라우드에 "BYON(bring your own notebook)" 서비스를 제공합니다. 버튼 클릭과 [DonkeyCar 노트북 이미지](https://github.com/AICoE/integration-demo-summit-2022) 에 대한 URL을 통해 데이터 과학자는 DonkeyCar 애플리케이션 및 종속 항목으로 완전히 구성된 전체 Jupyter 노트북 환경을 가동할 수 있습니다. 여기에서 데이터 과학자는 DonkeyCar 도구와 상호 작용하여 데이터를 생성하고, 실험을 실행하고, 결과를 그래프로 표시하고, 새로운 AI/ML 모델을 교육할 수 있습니다. 결과 및 변경 사항은 노트북이 생성된 기본 소스 리포지토리로 다시 커밋됩니다.

또한 DonkeyCar 애플리케이션 제품군은 가상 시뮬레이션에서 모델을 테스트하는 "디지털 트윈" 환경을 제공합니다. 데모에서 데이터 과학자는 DonkeyCar 애플리케이션 및 모델을 사용하여 AWS에서 서비스로 실행되는 시뮬레이션된 트랙 환경에 연결합니다. 데이터 과학자는 가상 자동차를 실행하여 모델의 성능을 테스트하고 새로운 데이터 포인트를 생성할 수 있습니다.



## 2. 공급망 보안

소프트웨어 공급망 보안은 독점 소프트웨어가 실제 행동과 결과를 도출하기 위해 입력을 해석해야 하는 AI 장치 에지 사용 사례에 특히 중요합니다. 또한 AI 모델을 구축하는 데이터 과학자는 클라우드 보안 전문가가 아닐 가능성이 높습니다. 클라우드 플랫폼은 소프트웨어 빌드 및 배포 절차를 자동화된 CI/CD 파이프라인으로 오프로드하여 데이터 과학자의 전문화된 역할을 충족할 수 있습니다. 이 자동화를 통해 애플리케이션 개발 배포의 여러 단계에서 보안 조치가 통합됩니다.

데모에서 DonkeyCar 애플리케이션을 빌드하고 배포하는 절차 는 데이터 과학자가 새 모델 버전을 커밋할 때 (Github 웹후크를 통해) 자동으로 트리거되는 [Tekton 파이프라인 으로 표현됩니다. ](https://cloud.redhat.com/blog/introducing-openshift-pipelines)두 개의 빌드 파이프라인이 병렬로 배포됩니다. 하나는 x86_64용이고 다른 하나는 ARM64용이며, 그 결과 자동차를 운전하는 데 필요한 소프트웨어를 번들로 묶는 컨테이너 이미지가 생성됩니다.

![img](https://next.redhat.com/wp-content/uploads/2022/08/image-1024x574.png)

빌드 파이프라인 내에서 [cosign ( ](https://github.com/sigstore/cosign)[SigStore](https://www.sigstore.dev/) 프로젝트 의 일부 )은 생성된 컨테이너 이미지의 암호화 서명을 캡처하는 데 사용됩니다. 이미지와 서명은 [Quay.io](https://quay.io/) 저장소에 함께 게시됩니다. 나중에 서명은 컨테이너 이미지와 함께 디바이스로 풀링되어 이미지가 배포되기 전에 Kubernetes 허용 컨트롤러에서 유효성을 검사할 수 있습니다. 올바르게 완료되면 서명을 사용하여 파이프라인 시작 시 빌드된 컨테이너 이미지가 배포 중인 것과 동일한지 확인합니다.

컨테이너 이미지의 검증을 신뢰하려면 먼저 검증자를 신뢰해야 합니다. 즉, 장치의 전체 소프트웨어 및 하드웨어 스택을 신뢰해야 합니다. 이를 위해 Red Hat Emerging Technology에서 시작된 보안 서비스인 [Keylime](https://next.redhat.com/project/keylime/) 을 사용하여 장치의 부팅 시간 증명 및 런타임 무결성을 제공할 수 있습니다. TPM(신뢰할 수 있는 플랫폼 모듈) 신뢰 루트를 사용하여 Keylime은 런타임 동안 시스템의 무결성을 모니터링하고 Linux IMA( [Integrity Measurement Architecture](https://www.redhat.com/en/blog/how-use-linux-kernels-integrity-measurement-architecture) ) 를 사용하여 수정된 파일 또는 설치된 패키지와 같은 불일치를 확인합니다 .



## 3. 에지 배포

검토를 위해 물리적 에지 장치에서 실행되는 세 가지 구성 요소인 Keylime, 이미지의 유효성을 검사하는 승인 컨트롤러 및 DonkeyCar 애플리케이션 자체입니다. 이를 실현하려면 이러한 각 구성 요소를 원격으로 배포하고 관리해야 합니다. 우리에게 필요한 것은 하이브리드 클라우드 컨트롤 플레인을 장치 자체로 확장하는 것입니다.

에지 운영의 상당한 발전은 임베디드 배포용으로 설계된 실험적인 단일 노드 OpenShift 인스턴스인 MicroShift입니다 [. ](https://github.com/openshift/microshift)MicroShift 는 기존 RHACM( [Red Hat Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management) ) 인스턴스에 연결하여 에지 장치를 대규모 하이브리드 클라우드 설치와 보다 원활하게 통합하여 Kubernetes 포드 및 서비스를 표준 OpenShift 도구를 사용하여 에지에 배포할 수 있도록 합니다. 데모에서 MicroShift를 사용하여 승인 컨트롤러와 애플리케이션 서비스를 배포합니다. 신뢰할 수 있는 배포를 보장하려면 MicroShift 서비스보다 먼저 장치에 Keylime을 설치하고 구성해야 합니다.

데모에서 에지 장치의 DonkeyCar 컨테이너 배포는 데이터 과학자의 커밋 푸시에 의해 트리거된 Tekton 파이프라인의 마지막 단계입니다. 이 마지막 단계에서는 애플리케이션이 실행될 위치를 지정하는 배포 매니페스트가 새 컨테이너 이미지 ID 및 Quay.io URL로 업데이트됩니다. 업데이트된 매니페스트는 ACM 배포에서 git 작업을 통해 모니터링 중인 Github 리포지토리로 푸시됩니다. 업데이트된 매니페스트를 읽으면 구독한 모든 에지 장치에 배포 작업이 푸시됩니다. 이 시점에서 컨테이너 이미지와 서명은 Quay.io에서 가져오고 서명은 승인 컨트롤러에 의해 확인되며 포함된 모드로 DonkeyCar 애플리케이션 컨테이너가 시작되어 자동차 작동이 시작됩니다.





DonkeyCar는 단순한 장난감과 같은 예를 제공하지만 시연된 인프라가 광범위한 장치 및 AI 기반 사용 사례에 적용될 수 있으며, 다양한 사용사례에 적합할 것으로 기대합니다. 이 게시물에서 설명하는 오픈 소스 기술이 곧 Red Hat 플랫폼과 고객에게 직접 적용될 것으로 기대됩니다.
