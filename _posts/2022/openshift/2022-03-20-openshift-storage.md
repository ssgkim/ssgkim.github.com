---
title: Kubernetes용 버킷 캐싱
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
description: 하이브리드 클라우드 상황에서 가장 이상적인 저장 수단을 고민해 본다.
article_tag1: OpenShift
article_tag2: Storage
article_tag3: DevOps
article_section: IT근황 공유하기
meta_keywords: openshift,odf,cloud,devsecops
last_modified_at: '2022-03-20 23:00:00 +0800'
---


### 스토리지 오브젝트 스토리지 시스템은 이기종 데이터 세트를 저장하는 간단하고 확장 가능하며 비용 효율적인 수단을 제공할 수 있다. 
#### 전통적으로 이러한 시스템은 미디어, 백업 및 아카이브용으로 설계되었습니다. 그러나 개체 스토리지 시스템의 사용은 데이터 및 AI 관련 애플리케이션과 기존 설계에 도전하는 하이브리드 클라우드 또는 에지와 같은 분산 환경을 포함하는 더 광범위한 워크플로로 점점 확장되고 있습니다.

데이터 및 AI 애플리케이션은 데이터에 대한 빈번하고 빠른 액세스를 필요로 하며 일반적으로 생성되면 변경할 수 없습니다. 분산 환경에서 일부 데이터는 로컬에서 생성되고 다른 데이터는 퍼블릭 클라우드 또는 데이터 센터와 같은 원격 위치에서 사용됩니다. 이러한 환경에 배포된 워크로드는 원격 네트워크를 통해 데이터를 전송할 때 성능이 저하될 수 있습니다. 또한 동일한 데이터에 반복적으로 액세스해야 하는 경우 대부분의 공용 클라우드에서 데이터 전송 또는 송신에 대해 반복적으로 요금이 부과됩니다.

Red Hat의 Emerging Technologies 블로그에는 업스트림 오픈 소스 커뮤니티와 Red Hat에서 활발히 개발 중인 기술에 대해 논의하는 게시물이 포함되어 있습니다. 우리는 우리가 작업하고 있는 것을 일찍 그리고 자주 공유할 것을 믿습니다. 그러나 달리 명시되지 않는 한 여기에서 공유되는 기술과 방법은 지원되는 제품의 일부가 아니며 미래에 약속되지도 않습니다.

#### 저장 위치 축소
일반적인 관행은 모든 위치에서 애플리케이션을 제공하기 위해 개체 저장소를 같은 위치에 배치하고 위치 간에 데이터를 동기화 및 보호하기 위한 복제를 사용하거나 사용자가 안팎으로 데이터를 복사해야 하는 완전히 자체 관리되는 방식으로 배치하는 것입니다. 로컬 스토리지. 그러나 이러한 로컬 개체 저장소는 관리 복잡성이 서서히 증가하기 시작하면서 격리된 저장소 사일로로 빠르게 실현됩니다.

여러 스토리지 위치를 관리하면 기업에 많은 문제가 발생합니다. 이는 스토리지 관리(용량, 암호화, 압축, 중복 제거, 안정성, 모니터링 등) 및 데이터 관리(권한, 복사, 버전 관리, 카탈로그, 분석 등)의 복잡성을 증가시킵니다. . 궁극적으로 분산 환경에서 데이터의 지역성은 성능상의 필요성이자 관리의 어려움입니다.

이러한 이유로 현대적인 디자인은 저장 위치를 ​​"허브"라고 하는 중앙 위치의 개체 저장소와 같은 "단일 소스 소스"로 축소하는 것을 의미합니다. 그러면 사용자와 해당 응용 프로그램이 네트워크를 통해 허브 버킷에 직접 액세스할 수 있지만 원격 네트워크와 로컬 저장소를 효율적이고 일관되게 사용하는 방법은 아직 설명하지 않습니다. 이러한 리소스의 사용을 조정하는 일반 플랫폼 서비스가 없으면 사용자는 애플리케이션이 처리할 수 있도록 로컬 스토리지에 데이터를 복사할 책임이 있습니다.

따라서 kubernetes 관리자를 위해 제안된 모델은 허브에 연결된 개체 저장소 게이트웨이를 제공하는 로컬 플랫폼 서비스를 배포하여 이러한 최적화의 복잡성을 원활하게 처리하는 동시에 모든 새로운 위치에서 로컬 저장소 관리를 자동화하는 것입니다. 이 모델은 게이트웨이가 후속 읽기를 위해 요청 시 허브에서 로컬 스토리지로 데이터를 가져오고 더 관련성 있는 데이터를 위한 공간을 만들기 위해 필요에 따라 콜드 데이터를 정리하는 캐시 패러다임을 따릅니다.

플랫폼 서비스는 또한 모든 로컬 워크로드에 클러스터 차원의 정책을 적용하는 이점이 있습니다. 이 정책은 각 워크로드가 고유한 정적 로컬 스토리지를 할당하는 대신 필요에 따라 실행 중인 워크로드 간에 로컬 스토리지 용량을 자동으로 공유할 수 있습니다. 이 모델을 염두에 두고 구현 방법을 살펴보겠습니다.

#### NooBaa의 버킷 캐싱
NooBaa 는 하이브리드 및 멀티 클라우드 환경을 위한 다양한 모델을 구현하는 유연한 개체 게이트웨이를 제공하는 오픈 소스 프로젝트입니다. 위에서 설명한 문제를 해결하기 위해 NooBaa는 버킷 캐싱 기능을 도입하고 있습니다. 이것이 어떻게 작동하는지 설명하려면 NooBaa에 대한 개요가 필요합니다.

일반적으로 NooBaa는 객체 저장소, 파일 시스템 및 블록 스토리지를 활용하는 다양한 하이브리드 정책을 버킷에 제공하기 위해 NooBaa 운영자의 Kubernetes CRD(CustomResourceDefinitions)를 사용하여 구성할 수 있는 S3 게이트웨이 서비스를 제공합니다. 이러한 정책을 정의하기 위해 NooBaa는 게이트웨이 버킷 정책을 구성하는 데 사용되는 기본 리소스 클래스인 "네임스페이스 리소스"(Kubernetes 네임스페이스와 혼동하지 말 것)라는 추상 개념을 사용합니다. NooBaa의 기본 네임스페이스 리소스 유형은 다음과 같습니다.

- 객체를 일부 백엔드 S3 호환 버킷으로 확인하는 "NS-S3".
- S3를 Azure Blob으로 변환하는 "NS-Blob".
- "NS-FS"는 파일 시스템을 S3 버킷으로 나타냅니다.
- "NS-Store"는 청크, 중복 제거, 압축, 암호화, 메타 데이터 관리 및 모든 스토리지(cloud/fs/block)에 데이터 청크 저장을 지원하는 네임스페이스입니다.
- "NS-Merge"는 이름/키 공간을 병합하여 여러 네임스페이스를 단일 버킷으로 결합합니다. 하이브리드 환경을 위한 새로운 기능을 도입하기 위해 네임스페이스 유형이 프로젝트에 추가되고 있습니다.

응용 프로그램에서 기본 리소스의 복잡성을 유지하는 새로운 데이터 관리 기능을 지원하기 위해 새로운 유형의 네임스페이스가 프로젝트에 추가되고 있습니다. 네임스페이스 버킷 기능에 대한 자세한 지침은 NooBaa 스택 다이어그램을 참조하세요.

"NS-Cache"는 캐싱 정책을 구현하는 새로운 네임스페이스입니다. 다른 모든 네임스페이스를 허브 네임스페이스(진실의 소스)로 사용할 수 있으며 NS-Store를 사용하여 캐싱을 위해 모든 유형의 로컬 스토리지를 사용할 수 있습니다. ). NooBaa 네임스페이스 추상화 위에 캐시 기능을 구축하여 지원되는 모든 스토리지를 허브 및 로컬 캐시에 채택할 수 있도록 최대한 유연하게 만듭니다.

#### 버킷 캐싱 작동 방식
애플리케이션은 상태 비저장, 경량 및 확장 가능한 포드인 NooBaa S3 엔드포인트를 사용하여 캐시 버킷과 상호 작용합니다. Kubernetes에 관심이 있는 경우 이러한 포드는 Kubernetes Deployment, Service 및 Horizontal Pod Autoscaler 개체와 OpenShift의 Route 개체에 의해 제어됩니다. 객체가 엔드포인트를 통해 캐시 버킷에 쓰거나 읽힐 때 NS-Cache 정책은 허브를 인수하여 허브를 사용하여 로컬 캐시 스토리지에서 사용할 수 없거나 이미 만료된 데이터를 쓰거나 읽고 객체로 로컬 캐시를 자동으로 업데이트합니다( 또는 물체의 일부).

초기 구현은 읽기 워크플로를 효율적으로 최적화하는 "연속 기입" 캐시 모델입니다. "write-through" 모드에서는 캐시 버킷에 생성된 새 객체가 허브에 직접 기록되는 동시에 객체의 로컬 복사본도 저장됩니다. 읽기 요청은 먼저 로컬 저장소에서 이행을 시도하고, 개체가 그곳에서 발견되지 않으면 허브에서 가져와서 TTL(Time to Live) 동안 로컬에 저장합니다. 사용 중인 개체(범위)의 관련 부분만 사용 가능한 용량 및 LRU를 기반으로 로컬로 유지됩니다.

캐시 모델을 사용하면 동일한 개체 또는 개체의 일부에 자주 액세스해야 하는 워크플로가 허브에서 개체를 반복적으로 가져올 필요가 없습니다. 또한, 객체의 해당 부분만을 로컬에 저장함으로써 로컬 스토리지에 상대적으로 적은 용량으로도 적중률을 최대화할 수 있도록 캐시의 효율성을 최적화한다.

#### 로드맵 및 요약
앞으로 버킷 캐싱 기능에 몇 가지 흥미로운 향후 개선 사항이 있습니다. 가장 주목할만한 것은 연결이 끊긴 환경을 허용하고 로컬 스토리지로 쓰기 속도를 높이는 "후기입" 캐시 모델을 구현하는 옵션입니다.

또 다른 영역은 정밀한 캐시 제어 힌트를 허용하여 더 많은 지식을 가진 사용자/관리자가 특정 데이터 세트에 대한 힌트를 미리 가져오거나 캐시에 고정할 수 있도록 합니다(즉, 퇴거 대상이 아님).

요약하면, 버킷 캐싱은 데이터 및 AI 애플리케이션에 필요한 효율성을 제공하는 동시에 클라우드 개체 저장소에서 사용할 수 있는 무한한 용량과 안정성을 활용하고 저장소 위치를 축소하는 데 도움이 될 수 있습니다.

사례출처 : https://github.com/noobaa/noobaa-core

{: .notice--info}

**참고자료** <br>
-- [https://github.com/noobaa/noobaa-core]({{"https://github.com/noobaa/noobaa-core"}}){:target="_blank"} <br>
{: .notice--info}

