---
title: gitops 프로모션 전략?
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
description: gitops를 사용하고자 할때 적절한 프로모션 전략은 무엇일까요?
article_tag1: OpenShift
article_tag2: SS-RH
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-06-17 23:00:00 +0800'
---



### Hello!! GitOps!

환경 간 승격을 GitOps 방식으로 구성하는 방법을 이해하는 데 어려움을 겪고 있습니다. 유즈케이스는 다소 간단하며 3개의 클러스터(dev, staging, prod)가 있습니다.

**Application X** 는 다음과 같은 방식으로 배포해야 하는 간단한 프로세스? 입니다.

```
1. auto-deploy to dev cluster (autosync)
2. trigger some smoke tests
3. auto-deploy to staging cluster
4. trigger smoke tests
5. manual approval
6. deploy to production
7. trigger smoke tests
```

애플리케이션 소스 코드와 Dockerfile은 `source.git`repo에 있습니다. 가 포함된 조타 장치 차트가 `values-<cluster>.yaml`repo에 `deploys.git`있습니다.

참고 사항:

- 모든 클러스터의 애플리케이션에 대해 자동 동기화가 켜져 있다는 점을 고려해야 합니다.. 즉, 마스터에 커밋하면 배포가 트리거됩니다.
- 클러스터당 분기를 사용하지 않고 대신 클러스터당 값 파일을 사용하고 마스터 분기를 감시합니다.
- 현재 app-of-apps 또는 ApplicationSet을 사용하지 않음
- ArgoCD는 단순성을 위해 3개의 클러스터에 설치되지만 클러스터 수가 증가하면 중앙 집중식으로 전환할 수 있습니다.

------

여기서 생각한 몇 가지 전략이 있지만 가장 좋은 방법이 무엇인지 약간 혼란스럽습니다.

### 전략 1: PostSync 작업이 원격 git 커밋을 수행합니다.

1. CI from 이 새 이미지 태그 `source.git`를 사용하여 원격 커밋합니다 .`deploys.git/application-x/values-dev.yaml`
2. ArgoCD는 dev에서 애플리케이션을 동기화합니다.
3. PostSync 작업이 연기 테스트를 실행합니다.
4. `deploys.git/application-x/values-staging.yaml`이미지 태그로 PostSync 작업 원격 커밋
5. ArgoCD는 스테이징 시 애플리케이션을 동기화합니다.
6. PostSync 작업이 연기 테스트를 실행합니다.
7. PostSync 작업은 `deploys.git/application-x/values-prod.yaml`누군가가 수동으로 승인하고 병합해야 하는 이미지 태그를 업데이트하기 위해 PR을 생성합니다.
8. 병합 시 ArgoCD는 제품의 애플리케이션을 동기화합니다.
9. PostSync 작업이 연기 테스트를 실행합니다.

보기에는 깔끔해 보이지만 이제 모든 애플리케이션 helm 차트에 동기화 후 원격 커밋 작업을 삽입해야 합니다. 또한 해당 작업에 대한 쓰기 액세스 권한을 부여하고 클러스터에서 git SSH 키를 관리해야 합니다.

### 전략 2: PostSync 작업이 커미터 워크플로를 트리거합니다.

이는 위와 동일하지만 사후 동기화 작업에서 원격 커밋을 수행할 때 이를 수행하는 워크플로를 트리거합니다. 따라서 위의 (4) 단계는 다음과 같습니다.

1. PostSync 작업은 원격 커밋을 수행하는 워크플로를 트리거합니다.`application-x/values-staging.yaml`

워크플로는 다음과 같은 매개변수를 사용하는 jenkins 파이프라인, argocd 워크플로, 일부 임시 CI 파이프라인 등이 될 수 있습니다.

```
serviceName: "application-x"
imageTag: "1.2.3"
env: "staging"
commitMessage: "auto-commit"
```

작업은 기본적으로 `deploys.git/application-x/values-staging.yaml`설정 `image.tag: 1.2.3`을 업데이트한 다음

```
git add application-x/values-staging.yaml
git commit -m ${commitMessage}
git push
```

유사하게 다른 작업은 PR 생성의 단계 (7)을 수행할 수 있습니다.

이 접근 방식은 커밋 논리를 외부 엔진에 위임하기 때문에 좋습니다. 그러나 이제 개발자는 이를 주시해야 합니다.

### 전략 3: 이미지 업데이터 패턴

이것은 Flux 1에 도입된 패턴입니다(지금은 Flux 2에서도 지원된다고 생각합니다).

이 패턴에서 컨트롤러는 업데이트를 위해 컨테이너 레지스트리(예: ECR)의 이미지 업데이트와 `image.tag`동기화를 트리거하는 git(git에 쓰기)의 업데이트(일부 정책 기반)를 감시합니다.

컨트롤러를 "이미지 업데이터"라고 부를 수 있습니다.

우리의 경우 다음과 같이 생각할 수 있습니다.

1. CI 파이프라인 푸시`application-x:dev`
2. 이미지 업데이터가 변경 사항을 감지하고 git이 새 SHA를 커밋합니다.`application-x/values-dev.yaml`
3. ArgoCD는 dev에서 동기화를 트리거하고 일부 연기 테스트를 실행합니다.
4. 완료되면 사후 동기화 작업은 로 태그 `application-x:dev`를 지정 `application-x:staging`하고 푸시 하는 워크플로를 트리거합니다.
5. 이미지 업데이터가 변경 사항을 감지하고 git이 새 SHA를 커밋합니다.`application-x/values-staging.yaml`
6. ArgoCD는 스테이징 시 동기화를 트리거하고 일부 연기 테스트를 실행합니다.
7. 완료되면 사후 동기화 작업은 태그 `application-x:staging`를 지정 `application-x:prod`하고 푸시하는 워크플로를 트리거합니다. 이 단계에서 PR을 생성하고 그렇게 할 수도 있습니다.
8. 프로덕션에 배포
9. 연기 테스트

이 접근 방식에서 이미지에 태그를 지정하는 것은 본질적으로 이미지를 홍보하는 것을 의미합니다.

###  상태

커뮤니티가 이 문제를 어떻게 공격하는지 확실히 확인 해야 할듯합니다.

동기화 후 워크플로/파이프라인을 트리거하는 경우 ArgoCD가 CD 프로세스의 "단계"가 됩니다. 그렇다면 Spinnaker와 같은 CD 도구는 전체 배포 중에 argocd sync가 "단계"가 될 수 있는 곳에서 더 합리적일까요?

{: .notice--info}

**참고자료** <br>
-- [https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/]({{"https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/"}}){:target="_blank"}<br>
{: .notice--info}
