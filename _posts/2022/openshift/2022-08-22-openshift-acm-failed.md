---
title: ACM 삭제를 실패 했을 경우
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
description: Operator를 통해 삭제를 실패한 경우
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-08-22 23:00:00 +0800'
---

# ACM 삭제를 실패 했을 경우 - CRD multiclusterhub를 제거할 수 없음





## 환경

- OpenShift 컨테이너 플랫폼 4.x
- Kubernetes 2.x용 Red Hat ACM

## 문제

- ACM multiclusterhub 인스턴스를 제거하지 못했습니다.
- ACM multiclusterhub-operator를 제거하지 못했습니다.
- ACM multiclusterhub-operator를 제거하려고 할 때 CRD multiclusterhub의 상태가 제거 중에서 멈춤

## 해결

- crd/multiclusterhubs.operator.open-cluster-management.io의 종료자를 null로 패치합니다.



```
oc patch crd/multiclusterhubs.operator.open-cluster-management.io -p '{"metadata":{"finalizers":[]}}' --type=merge
```

- multiclusterhub 다시 삭제



```
oc delete  multiclusterhubs.operator.open-cluster-management.io -n open-cluster-management   multiclusterhub
```

- multiclusterhub가 삭제되었는지 확인하십시오.



```
oc get  multiclusterhubs.operator.open-cluster-management.io -n open-cluster-management   multiclusterhub
```

## 근본 원인

- 고객이 실수로 ACM multiclusterhub 인스턴스를 제거하기 전에 가져온 클러스터를 먼저 제거하는 올바른 절차를 따르지 않았습니다.
- 종료자는 crd/multiclusterhubs.operator.open-cluster-management.io를 차단하여 삭제 프로세스를 완료합니다.

## 진단 단계

1. oc get multiclusterhubs.operator.open-cluster-management.io -n open-cluster-management multiclusterhub
   제거에서 리소스가 멈춘 것을 볼 수 있습니다.
2. webhook 서비스가 실수로 삭제되어 multiclusterhubs.operator.open-cluster-management.io/multiclusterhub의 종료자를 지우지 못했습니다.

```
$ oc patch  multiclusterhubs.operator.open-cluster-management.io -n open-cluster-management   multiclusterhub --type=merge -p '{"metadata": {"finalizers":[]}}'
Error from server (InternalError): Internal error occurred: failed calling webhook "multiclusterhub.validating-webhook.open-cluster-management.io": Post "https://multiclusterhub-operator-webhook.open-cluster-management.svc:443/validate-v1-multiclusterhub?timeout=10s": service "multiclusterhub-operator-webhook" not found
```


{: .notice--info}

**참고자료** <br>
-- [https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators]({{"https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators"}}){:target="_blank"}<br>
{: .notice--info}
