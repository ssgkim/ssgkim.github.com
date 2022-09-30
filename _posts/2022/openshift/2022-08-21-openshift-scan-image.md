---
title: openshift에서 이미지 스캐닝을 위한 바이러스 테스트 수행
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
description: 도커 이미지 보안 스캐닝을 위한 바이러스 테스트
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-08-21 23:00:00 +0800'
---

# 도커 이미지 보안 스캐닝을 위한 바이러스 테스트

거의 모든 컨테이너 플랫폼에는 유명한 clair와 같은 컨테이너 보안 솔루션이 있지만 스캔 원칙은 심층 스캔이 아니지만 컨테이너 내부의 yum 및 apk와 같은 패키지 관리 도구를 통해 패키지 관리 도구의 과거 데이터베이스를 스캔합니다. 소프트웨어 버전 참조 취약점이 있는지 여부를 확인하기 위해 컨테이너에 설치됩니다.

물론 이 스캐닝 방식은 성능을 위한 것이지만 일상생활에 지장을 주기도 합니다.일반적으로 엔지니어는 컨테이너 보안 플랫폼이 있으면 편히 쉬고 쉴 수 있다고 생각하기 쉽지만 사실은 그렇지 않습니다.

아래의 실제 예를 들어 그 효과를 확인한 다음 이를 처리하는 방법에 대해 생각해 보겠습니다.

# 테스트 키/클레어/도커 허브

온라인 테스트 바이러스를 사용하여 컨테이너에 복사하고 패키징하여 미러웨어하우스에 업로드하고 미러웨어하우스의 스캔 결과를 확인합니다. 좀 더 대표성을 높이기 위해 바이러스를 자바에 복사한 후 이미지를 docker 허브인 quay.io에 업로드합니다.

```bash
mkdir -p /data/tmp
cd /data/tmp

cat << EOF > ./virus.Dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal
ADD https://www.ikarussecurity.com/wp-content/downloads/eicar_com.zip /wzh
ADD https://github.com/MalwareSamples/Linux-Malware-Samples/blob/main/00ae07c9fe63b080181b8a6d59c6b3b6f9913938858829e5a42ab90fb72edf7a /wzh01
ADD https://github.com/MalwareSamples/Linux-Malware-Samples/blob/main/00ae07c9fe63b080181b8a6d59c6b3b6f9913938858829e5a42ab90fb72edf7a /usr/bin/java

RUN chmod +x /wzh*
RUN chmod +x /usr/bin/java
EOF

buildah bud -t quay.io/wangzheng422/qimgs:virus -f virus.Dockerfile ./

buildah push quay.io/wangzheng422/qimgs:virus

buildah bud -t docker.io/wangzheng422/virus -f virus.Dockerfile ./

buildah push docker.io/wangzheng422/virus
```

바이러스가 포함된 이미지는 두 컨테이너 이미지 플랫폼에서 스캔할 수 없음을 발견했습니다. 이것은 또한 일반 검색 도구가 패키지 관리 도구의 이력 데이터베이스만 검색할 수 있고 컨테이너 내부의 소프트웨어 버전은 검색할 수 없다는 것을 증명합니다.

# log4jshell

이제 Dingding의 log4j 허점에 대해 다른 플랫폼은 다른 플랫폼을 스캔할 수 있습니다.스캔할 수 있는 것은 컨테이너에서 jar 파일을 감지한 다음 jar 파일에서 MANIFEST.MF 파일을 보고 버전을 확인하는 것입니다. 이 파일에 있는 해당 소프트웨어 패키지 번호를 누르고 경찰에 신고하십시오.

quay.io/apoczeka/log4shell-vuln 컨테이너를 보면 quay.io에서 스캔한 결과 log4j 취약점을 찾을 수 없었습니다.

## ACS

그런 다음 Red Hat RHACS 컨테이너 보안 플랫폼이 이를 제거할 수 있는지 봅시다.

```bash
# on vultr
wget https://mirror.openshift.com/pub/rhacs/assets/latest/bin/linux/roxctl
install -m 755 roxctl /usr/local/bin/

# on ACS platform
# Integrations -> API Token -> Create Integration
# role -> continous-integration -> create
# copy the API token out
export ROX_API_TOKEN=<api_token>
export ROX_CENTRAL_ADDRESS=central-stackrox.apps.cluster-ms246.ms246.sandbox1059.opentlc.com:443

roxctl -e "$ROX_CENTRAL_ADDRESS" --insecure-skip-tls-verify image scan -i docker.io/elastic/logstash:7.13.0 | jq '.scan.components[] | .vulns[]? | select(.cve == "CVE-2021-44228") | .cve'
# "CVE-2021-44228"
# "CVE-2021-44228"

roxctl -e "$ROX_CENTRAL_ADDRESS" --insecure-skip-tls-verify image scan -i quay.io/apoczeka/log4shell-vuln | jq '.scan.components[] | .vulns[]? | select(.cve == "CVE-2021-44228") | .cve'
# "CVE-2021-44228"

roxctl -e "$ROX_CENTRAL_ADDRESS" --insecure-skip-tls-verify image check -r 0 -o json -i docker.io/elastic/logstash:7.13.0 
```

ACS가 log4j 취약점을 성공적으로 감지했음을 알 수 있습니다. 이러한 방식으로 ACS를 CI/CD 파이프라인으로 상속하여 취약점 검색을 완료할 수 있습니다.

물론 ACS의 내장 인터페이스 도구를 사용하여 규칙을 신속하게 정의하고 관련 취약점을 처음으로 방지할 수 있습니다.

![img](https://wangzheng422.github.io/docker_env/notes/2021/imgs/2021-12-17-16-32-12.png)

다음은 구성이 적용된 후 취약한 이미지가 실행되지 않도록 하는 ACS의 효과입니다(클러스터가 사용 중인 경우 전달하는 데 약간의 시간이 소요됨).

![img](https://wangzheng422.github.io/docker_env/notes/2021/imgs/2021-12-17-16-34-15.png)

그러나 현재 ACS는 배포 모드에서만 배포를 지원합니다. 배포를 수정하거나 단순히 포드를 사용하여 직접 배포하면 ACS 감지가 우회됩니다. 이 문제는 향후 ACS 업그레이드로 해결될 것입니다.

# 그래프

ACS와 같은 명령줄 도구에는 다른 많은 옵션이 있습니다. 여기에 예가 있습니다.

```bash
# https://github.com/anchore/grype

grype -q quay.io/apoczeka/log4shell-vuln | grep log4j
# log4j-api          2.14.1       2.15.0       GHSA-jfh8-c2jp-5v3q  Critical
# log4j-api          2.14.1       2.16.0       GHSA-7rjr-3q55-vv33  Medium
# log4j-api          2.14.1                    CVE-2021-44228       Critical
# log4j-core         2.14.1       2.15.0       GHSA-jfh8-c2jp-5v3q  Critical
# log4j-core         2.14.1       2.16.0       GHSA-7rjr-3q55-vv33  Medium
# log4j-core         2.14.1                    CVE-2021-44228       Critical
# log4j-jul          2.14.1                    CVE-2021-44228       Critical
# log4j-slf4j-impl   2.14.1                    CVE-2021-44228       Critical
```


{: .notice--info}

**참고자료** <br>
-- [https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators]({{"https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators"}}){:target="_blank"}<br>
{: .notice--info}
