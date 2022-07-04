---
title: 클러스터에서 오퍼레이터 삭제 방법
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
description: 오퍼레이터 삭제가 잘안될때, 확인하기
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-07-05 23:00:00 +0800'
---

### 클러스터에서 오퍼레이터 삭제

- [웹 콘솔을 사용하여 클러스터에서 오퍼레이터 삭제](https://docs.openshift.com/container-platform/4.10/operators/admin/olm-deleting-operators-from-cluster.html#olm-deleting-operators-from-a-cluster-using-web-console_olm-deleting-operators-from-a-cluster)
- [CLI를 사용하여 클러스터에서 오퍼레이터 삭제](https://docs.openshift.com/container-platform/4.10/operators/admin/olm-deleting-operators-from-cluster.html#olm-deleting-operator-from-a-cluster-using-cli_olm-deleting-operators-from-a-cluster)
- [실패한 구독 새로 고침](https://docs.openshift.com/container-platform/4.10/operators/admin/olm-deleting-operators-from-cluster.html#olm-refresh-subs_olm-deleting-operators-from-a-cluster)

다음은 OpenShift Container Platform 클러스터에서 OLM(Operator Lifecycle Manager)을 사용하여 이전에 설치된 Operator를 삭제하는 방법을 설명합니다.

### 웹 콘솔을 사용하여 클러스터에서 오퍼레이터 삭제

클러스터 관리자는 웹 콘솔을 사용하여 선택한 네임스페이스에서 설치된 오퍼레이터를 삭제할 수 있습니다.

전제 조건

- 권한이 있는 계정을 사용하여 OpenShift Container Platform 클러스터 웹 콘솔에 액세스합니다 `cluster-admin`.

절차

1. **운영자** → **설치된 운영자** 페이지 로 이동합니다 .

2. **필터 이름** 필드 에 키워드를 스크롤하거나 입력하여 제거하려는 오퍼레이터를 찾습니다. 그런 다음 클릭하십시오.

3. **오퍼레이터 세부 정보** 페이지 의 오른쪽에 있는 **작업** 목록 에서 **오퍼레이터 제거 를 선택합니다.**

   제거 오퍼레이터 **?** 대화 상자가 표시됩니다.

4. 오퍼레이터, 오퍼레이터 배포 및 팟(Pod)을 제거하려면 **제거** 를 선택하십시오 . 이 작업 후에 오퍼레이터는 실행을 중지하고 더 이상 업데이트를 수신하지 않습니다.

   |      | 이 작업은 사용자 지정 리소스 정의(CRD) 및 사용자 지정 리소스(CR)를 포함하여 오퍼레이터가 관리하는 리소스를 제거하지 않습니다. 웹 콘솔에서 활성화된 대시보드 및 탐색 항목과 계속 실행되는 클러스터 외부 리소스는 수동 정리가 필요할 수 있습니다. Operator를 제거한 후 이를 제거하려면 Operator CRD를 수동으로 삭제해야 할 수 있습니다. |
   | ---- | ------------------------------------------------------------ |
   |      |                                                              |

### CLI를 사용하여 클러스터에서 운영자 삭제

클러스터 관리자는 CLI를 사용하여 선택한 네임스페이스에서 설치된 오퍼레이터를 삭제할 수 있습니다.

전제 조건

- 권한이 있는 계정을 사용하여 OpenShift Container Platform 클러스터에 액세스합니다 `cluster-admin`.
- `oc`워크스테이션에 설치된 명령.

절차

1. 필드 에서 등록된 연산자의 현재 버전(예 : `jaeger`)을 확인합니다.`currentCSV`

   

   ```
   $ oc get subscription jaeger -n openshift-operators -o yaml | grep currentCSV
   ```

   예시 출력

   

   ```
     currentCSV: jaeger-operator.v1.8.2
   ```

2. 구독 삭제(예 `jaeger`: ):

   

   ```
   $ oc delete subscription jaeger -n openshift-operators
   ```

   예시 출력

   

   ```
   subscription.operators.coreos.com "jaeger" deleted
   ```

3. `currentCSV`이전 단계의 값을 사용하여 대상 네임스페이스에서 오퍼레이터에 대한 CSV를 삭제합니다 .

   

   ```
   $ oc delete clusterserviceversion jaeger-operator.v1.8.2 -n openshift-operators
   ```

   예시 출력

   

   ```
   clusterserviceversion.operators.coreos.com "jaeger-operator.v1.8.2" deleted
   ```

## 실패한 구독 새로 고침

OLM(Operator Lifecycle Manager)에서 네트워크에서 액세스할 수 없는 이미지를 참조하는 Operator에 가입하면 `openshift-marketplace`네임스페이스에서 다음 오류와 함께 실패한 작업을 찾을 수 있습니다.

예시 출력



```
ImagePullBackOff for
Back-off pulling image "example.com/openshift4/ose-elasticsearch-operator-bundle@sha256:6d2587129c846ec28d384540322b40b05833e7e00b25cca584e004af9a1d292e"
```

예시 출력



```
rpc error: code = Unknown desc = error pinging docker registry example.com: Get "https://example.com/v2/": dial tcp: lookup example.com on 10.0.0.1:53: no such host
```

결과적으로 구독은 이 실패 상태에서 멈추고 오퍼레이터는 설치 또는 업그레이드할 수 없습니다.

구독, CSV(클러스터 서비스 버전) 및 기타 관련 개체를 삭제하여 실패한 구독을 새로 고칠 수 있습니다. 구독을 다시 만든 후 OLM은 올바른 버전의 Operator를 다시 설치합니다.

전제 조건

- 액세스할 수 없는 번들 이미지를 가져올 수 없는 실패한 구독이 있습니다.
- 올바른 번들 이미지에 액세스할 수 있음을 확인했습니다.

절차

1. Operator가 설치된 네임스페이스에서 `Subscription`및 개체 의 이름을 가져옵니다.`ClusterServiceVersion`

   

   ```
   $ oc get sub,csv -n <namespace>
   ```

   예시 출력

   

   ```
   NAME                                                       PACKAGE                  SOURCE             CHANNEL
   subscription.operators.coreos.com/elasticsearch-operator   elasticsearch-operator   redhat-operators   5.0
   
   NAME                                                                         DISPLAY                            VERSION    REPLACES   PHASE
   clusterserviceversion.operators.coreos.com/elasticsearch-operator.5.0.0-65   OpenShift Elasticsearch Operator   5.0.0-65              Succeeded
   ```

2. 구독 삭제:

   

   ```
   $ oc delete subscription <subscription_name> -n <namespace>
   ```

3. 클러스터 서비스 버전을 삭제합니다.

   

   ```
   $ oc delete csv <csv_name> -n <namespace>
   ```

4. 네임스페이스 에서 실패한 작업 및 관련 구성 맵의 이름을 가져옵니다 `openshift-marketplace`.

   

   ```
   $ oc get job,configmap -n openshift-marketplace
   ```

   예시 출력

   

   ```
   NAME                                                                        COMPLETIONS   DURATION   AGE
   job.batch/1de9443b6324e629ddf31fed0a853a121275806170e34c926d69e53a7fcbccb   1/1           26s        9m30s
   
   NAME                                                                        DATA   AGE
   configmap/1de9443b6324e629ddf31fed0a853a121275806170e34c926d69e53a7fcbccb   3      9m30s
   ```

5. 작업 삭제:

   

   ```
   $ oc delete job <job_name> -n openshift-marketplace
   ```

   이렇게 하면 액세스할 수 없는 이미지를 가져오려는 포드가 다시 생성되지 않습니다.

6. 구성 맵을 삭제합니다.

   

   ```
   $ oc delete configmap <configmap_name> -n openshift-marketplace
   ```

7. 웹 콘솔에서 OperatorHub를 사용하여 Operator를 다시 설치합니다.

확인

- Operator가 성공적으로 다시 설치되었는지 확인합니다.

  

  ```
  $ oc get sub,csv,installplan -n <namespace>
  ```
  
  
{: .notice--info}

**참고자료** <br>
-- [https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators]({{"https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators"}}){:target="_blank"}<br>
{: .notice--info}
