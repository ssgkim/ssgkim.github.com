---
title: 안전한 파이프라인 구축
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
description: 안전한 파이프 라인이란
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-12-21 17:00:00 +0800'

---





# 안전한 파이프라인 구축



보안에 대한 위협을 어떻게 해결할까? 에 대한 물음은 조금더 적극적인 방법으로 그 솔루션을 찾고 있습니다.

그중 가장 보편적이고 안전한 방법인 파이프라인을 통한 강화 방법에 대해 확인해 보도록 하겠습니다.

이 게시물에서는 자체 빌드 파이프라인을 강화하는 방법과 소프트웨어를 보다 책임감 있게 사용하는 방법에 대해 설명합니다. [오픈 소스 도구 Podman](https://podman.io/) [Cosign](https://github.com/sigstore/cosign) , [Syft](https://github.com/anchore/syft),  [Grype](https://github.com/anchore/grype) 을 사용하여 서명, 소프트웨어 BOM, 스캐닝 및 증명을 통합하는 방법을 소개합니다.





## 범용 기본 이미지

소프트웨어 파이프라인을 보호하기 위한 전제 조건은 신뢰할 수 있는 콘텐츠를 기반으로 구축하는 것입니다. 업계가 컨테이너화된 애플리케이션 제공으로 전환했을 때 Red Hat은 **루트가 아닌 사용자로** 실행 되는 **신뢰할 수 있는 소스** **의 최소 이미지** 를 제공하기 시작했습니다 . 이러한 보안 기능은 신뢰할 수 있는 소프트웨어를 생산할 때 필수적인 첫 번째 단계입니다. RHEL 9 (UBI9) [기반의 범용 기본 이미지](https://catalog.redhat.com/software/containers/search?q=ubi9&p=1&product_listings_names=Red Hat Universal Base Image 9) 를 예로 들어 보겠습니다 . 다양한 사용 사례에는 UBI9, UBI9-minimal 및 UBI9-micro와 같은 광범위한 기본 이미지 세트가 필요합니다. 여기에는 [신뢰할 수 있는 패키지의 기본 집합 이 포함됩니다.](https://catalog.redhat.com/software/containers/ubi9-micro/61832b36dd607bfc82e66399?container-tabs=packages), 사용자가 애플리케이션을 구축할 수 있는 RHEL9와 동일한 파이프라인에서. 레지스트리 콘솔에는 이미지 내의 버전, 서명, [Clair](https://quay.github.io/clair/) 를 통한 취약점 스캔 결과 , 이미지 버전, Dockerfile, 이미지 다운로드 방법 및 기타 기술 정보와 함께 패키지 목록이 포함됩니다. 다음 스크린샷은 레지스트리 콘솔에서 UBI9 이미지의 보안 프로필을 보여줍니다. 취약성 관리에 대한 UBI의 위험 기반 접근 방식에 대한 자세한 내용은 [여기 에서 확인할 수 있습니다.](https://access.redhat.com/sites/default/files/pages/attachments/an_open_approach_to_vulnerability_management_v1.4.pdf)

![UBI 9 Minimal용 Red Hat 카탈로그 페이지 항목 - 안전한 빌드 파이프라인을 위한 기본 컨테이너 이미지](https://next.redhat.com/wp-content/uploads/2022/10/Screen-Shot-2022-10-27-at-09.21.30-1024x829.png)

UBI 기본 이미지는 다단계 빌드에 적합합니다. 다단계 빌드를 사용하면 결과 이미지에 실행에 필요한 항목만 포함됩니다. 그 이유는 이미지에 포함된 패키지가 적을수록 공격 표면이 작아지기 때문입니다. 포함된 패키지 관리자가 없기 때문에 [UBI-micro 기본 이미지 가 가장 작습니다(패키지를 추가하려면 ](https://www.redhat.com/en/blog/introduction-ubi-micro)[Buildah](https://github.com/containers/buildah) 및 컨테이너 호스트의 패키지 관리자에 의존해야 함). *UBI-micro 이미지는 distroless* 컨테이너 이미지 라고도 합니다 . 소프트웨어 빌드 시스템에 Red Hat의 UBI 이미지를 포함하는 것을 고려하십시오. 이러한 최소한의 인증되고 신뢰할 수 있는 기본 이미지는 안전한 소프트웨어 파이프라인을 위한 토대를 제공합니다.



## 소프트웨어 재료 명세서(SBOM)란 무엇일까?

SBOM(Software Bill of Materials)을 구성하는 세부 사항은 현재 결정 중이며 최종적으로 표준화될 예정입니다. 오늘날 위험 관리를 최우선으로 고려하여 어떻게 이미지를 구축하고 사용할 수 있습니까? 현재 이미지에 구성 요소 매니페스트를 포함하지 않은 경우 빌드 프로세스에 추가하는 것이 좋습니다. 이를 통해 커뮤니티에서 소프트웨어 BOM이 정확히 무엇인지 파악하는 동안 작업할 프로세스를 구축할 수 있습니다. 일반적으로 "SBOM"은 소프트웨어에 포함된 버전과 함께 패키지 및 구성 요소를 설명합니다. 이러한 매니페스트를 생성하는 여러 오픈 소스 도구가 있습니다. [Syft](https://github.com/anchore/syft) 는 다음 예제에서 사용됩니다. Syft는 Sigstore/Cosign 및 컨테이너 레지스트리 [Quay.io](https://quay.io/repository/) 와 원활하게 작동합니다 .

SBOM은 귀하가 받은 코드가 릴리스된 코드라는 증명이나 보증 없이는 큰 가치를 갖지 못합니다. 증명은 조건자로 알려진 이벤트 또는 아티팩트의 무결성을 확인하는 데 사용되는 암호화 서명된 메타데이터입니다. 이 경우 SBOM은 술어이고 증명은 SBOM 내의 코드를 확인하는 메타데이터입니다. SBOM이 있는 증명은 빌드 프로세스의 일부로 생성되어 SBOM이 이미지에 첨부되기 전에 변조되지 않았는지 확인해야 합니다. 다음 예에서는 SBOM과 함께 [in-toto 증명 이 생성됩니다. ](https://github.com/in-toto/in-toto)[인토토](https://in-toto.io/)소프트웨어 공급망의 적법성과 무결성을 검증하기 위한 개방형 메타데이터 표준 및 프레임워크입니다. 다행스럽게도 Podman, Cosign, Syft, Grype 및 Quay.io와 같은 도구를 사용하면 이미지에 SBOM 및 증명을 쉽게 첨부할 수 있습니다.



## 무엇을 기다리고 있나요? 

Red Hat은 산업 전반에 걸쳐 SBOM 에코시스템의 다른 업체와 협력하여 SBOM을 대규모로 제공하기 위한 표준을 마련하고 있습니다. 도구와 표준은 진화하고 있습니다. 여기에는 귀하가 생산하고 소비하는 소프트웨어의 보안 및 무결성을 개선하기 위해 현재 소프트웨어 파이프라인에 통합해야 하는 조치가 요약되어 있습니다.



### 전제 조건

[Podman](https://podman.io/getting-started/installation) , [Cosign](https://github.com/sigstore/cosign#installation) 및 [Syft](https://github.com/anchore/syft#recommended) 를 설치 하고 [Quay.io](https://quay.io/repository/) 로 계정을 생성합니다 .

podman `quay.io/sallyom/hello-go:test`을 사용하여 이미지를 빌드하고 푸시했습니다. Dockerfile 은 권한이 없는 사용자 를 기반으로 한 결과로 다단계 빌드를 지정합니다 [. ](https://github.com/sallyom/hello-go/blob/main/Dockerfile)`UBI9-micro`Cosign을 사용하여 Sigstore ECDSA-P256 이미지 서명 키 쌍을 생성합니다. Sigstore의 키 쌍 형식에 대한 자세한 내용은 [여기](https://blog.sigstore.dev/cosign-image-signatures-77bab238a93) 를 참조하십시오 .

```bash
$ cosign generate-key-pair
# add Password if desired
Private key written to cosign.key
Public key written to cosign.pub
```



### Sigstore 키 쌍으로 서명

이미지에 서명합니다. 여기에서는 Cosign이 사용되지만 Podman은 Sigstore/Cosign 키 쌍으로 서명할 수도 있습니다. Podman은 최근에 `podman push –sign-with-sigstore-private-key`. Podman 버전 4.2부터 사용자는 Sigstore 서명에 서명, 첨부 및 푸시하도록 선택할 수 있습니다.

```bash
$ cosign sign --key ~/cosign.key quay.io/sallyom/hello-go:test
Enter password for private key:
Pushing signature to: quay.io/sallyom/hello-go
```



### SBOM 생성

이미지에 대한 소프트웨어 BOM을 생성합니다. Syft에서 정의한 SBOM은 [여기에서](https://github.com/sallyom/hello-go/blob/5db9871c71d4ea1e3372fd3cdb5011c42f63dfdb/hello-go.sbom) 볼 수 있습니다 .

```bash
$ syft -o spdx podman:quay.io/sallyom/hello-go:test > hello-go.sbom
 ✔ Loaded image            
 ✔ Parsed image            
 ✔ Cataloged packages      [23 packages]
```



### SBOM 서명, 첨부 및 증명

SBOM 조건자에 대한 증명에 서명하고 첨부합니다. 이를 통해 SBOM은 증명 내에 있는 그대로 서명(따라서 변조 방지)되고 소비자는 진위를 확인할 수 있습니다. [이 문제](https://github.com/sigstore/cosign/issues/1742) 는 attach 및 attest 명령 간의 차이점을 설명합니다. Cosign 명령에 추가된 –verbose 플래그는 여기에 제공된 출력보다 더 자세한 정보를 나타냅니다.

```bash
$ cosign sign --key ~/cosign.key --attachment sbom quay.io/sallyom/hello-go:test
Enter password for private key: 
Pushing signature to: quay.io/sallyom/hello-go

$ cosign attach sbom --sbom hello-go.sbom quay.io/sallyom/hello-go:test
Uploading SBOM file for [quay.io/sallyom/hello-go:test] to [quay.io/sallyom/hello-go:sha256-a66f3923d0ea7890bce8586690f1fbfbcf3ae1d0935acc372299b29c99f92e50.sbom] with mediaType [text/spdx].

$ cosign attest --key ~/cosign.key --predicate hello-go.sbom quay.io/sallyom/hello-go:test
Enter password for private key: 
Using payload from: hello-go.sbom
```



개인 키 **의** 비밀번호 입력 :

다음에서 페이로드 사용: hello-go. 스봄

서명, SBOM 및 증명을 이미지에 첨부하는 것은 몇 가지 명령만큼 간단합니다. 소프트웨어 소비자로서 Podman, Cosign 및 Quay는 귀하가 의도한 대로 정확하게 실행되도록 보장할 수 있습니다. 다음은 컨테이너화된 워크로드의 무결성과 보안을 보장하기 위한 몇 가지 제안 사항입니다.

### 다운로드하기 전에 서명된 이미지 확인

Podman을 사용하면 서명되지 않은 이미지를 가져올 수 없도록 정책을 설정할 수 있으며 Sigstore로 서명해야 한다고 지정할 수도 있습니다. Podman의 policy.json 파일에 다음을 추가하십시오.

```yaml
# for root podman the file is /etc/containers/policy.json
$ cat ~/.config/containers/policy.json 
{
    ---
    "transports": {
        "docker": {
            "quay.io/sallyom": [
                {
                    "type": "sigstoreSigned",
                    "keyPath": "/home/somalley/cosign.pub",
                    "signedIdentity": {"type": "matchRepository"}
                }
            ],
```



이렇게 하면 첨부된 서명이 없으면 quay.io/sallyom/image 가져오기가 실패합니다.

```yaml
$ podman pull quay.io/sallyom/cli:test
Trying to pull quay.io/sallyom/cli:test...
Error: Source image rejected: A signature was required, but no signature exists

$ podman pull quay.io/sallyom/hello-go:test
Trying to pull quay.io/sallyom/hello-go:test...
Getting image source signatures
Checking if image destination supports signatures
----- 
Copying config 93e70746a6 done  
Writing manifest to image destination
Storing signatures
93e70746a69c780df7a1961487c7b77726eacadd3f70f6f960d2320580968eab
```



### 이미지에 대해 서명된 SBOM 확인

```yaml
$ cosign verify --key ~/cosign.pub --attachment sbom quay.io/sallyom/hello-go:test

Verification for quay.io/sallyom/hello-go:sha256-a66f3923d0ea7890bce8586690f1fbfbcf3ae1d0935acc372299b29c99f92e50.sbom --
```



이러한 각 서명 **에** 대해 다음 검사가 수행되었습니다 .

  \- 코사인 클레임이 확인되었습니다.

  \- 지정된 공개 키에 대해 서명이 확인되었습니다.



### SBOM 진위 증명 확인

```bash
$ cosign verify-attestation --key ~/cosign.pub quay.io/sallyom/hello-go:test > /tmp/att
```



이러한 각 서명 **에** 대해 다음 검사가 수행되었습니다 .

  \- 코사인 클레임이 확인되었습니다.

  \- 지정된 공개 키에 대해 서명이 확인되었습니다.





### 취약점에 대한 이미지 스캔

Grype는 이미지에서 알려진 취약점을 스캔하는 도구입니다. 이 도구는 데이터 소스를 추가하도록 구성할 수 있지만 데이터 [를 가져오는 위치에 대한 적절한 기본값이 있습니다](https://github.com/anchore/grype#grypes-database) . Podman으로 Grype 이미지를 실행할 수 있으므로 Grype를 설치할 필요가 없습니다. Podman 소켓이 실행 중이지 않은 경우(서비스가 Podman과 함께 설치되었지만 활성화되지 않음) 다음을 사용하여 시작할 수 있습니다.

`$ systemctl --user start podman.socket`



```bash
$ podman run --rm --volume /run/user/1000/podman/podman.sock:/var/run/docker.sock --name Grype anchore/grype:latest quay.io/sallyom/hello-go:test
NAME          INSTALLED           FIXED-IN     TYPE  VULNERABILITY   SEVERITY 
libgcc        11.2.1-9.4.el9                   rpm   CVE-2021-46195  Low       
libgcc        11.2.1-9.4.el9      (won't fix)  rpm   CVE-2022-27943  Low       
ncurses-base  6.2-8.20210508.el9  (won't fix)  rpm   CVE-2022-29458  Low       
ncurses-libs  6.2-8.20210508.el9  (won't fix)  rpm   CVE-2022-29458  Low
```



마지막으로 Cosign에는 이미지의 알려진 보안 관련 아티팩트를 나열하는 올인원 명령이 있습니다. 

명령은 `cosign tree`.

```yaml
$ cosign tree quay.io/sallyom/hello-go:test
📦 Supply Chain Security Related artifacts for an image: quay.io/sallyom/hello-go:test
└── 💾 Attestations for an image tag: quay.io/sallyom/hello-go:sha256-a66f3923d0ea7890bce8586690f1fbfbcf3ae1d0935acc372299b29c99f92e50.att
   ├── 🍒 sha256:06a8bd80b513e9f98009cddd4856f9e54c22cba3df2a9714b457e8c10b1c0b5c
   ├── 🍒 sha256:06a8bd80b513e9f98009cddd4856f9e54c22cba3df2a9714b457e8c10b1c0b5c
   └── 🍒 sha256:22dfc2808a9008d76c79ef0b80cc06a569c367fe880fbf155600385125e18b4a
└── 🔐 Signatures for an image tag: quay.io/sallyom/hello-go:sha256-a66f3923d0ea7890bce8586690f1fbfbcf3ae1d0935acc372299b29c99f92e50.sig
   └── 🍒 sha256:94683d70b9542b4e5a1d9be2e27923f9f37e3a7fc37c3900b0ff90a6be3c8e9c
└── 📦 SBOMs for an image tag: quay.io/sallyom/hello-go:sha256-a66f3923d0ea7890bce8586690f1fbfbcf3ae1d0935acc372299b29c99f92e50.sbom
   └── 🍒 sha256:87ee673638aaf1c194241b65794a508fa02f93503e1113d4e6eb8278dd28bfdf
```



오픈 소스와 커뮤니티 주도 코드에 의존하며 안전하고 선별가능한 공급망 보안을 위한 방법을 확인했습니다. 소프트웨어 공급망의 보안을 강화하는 것은 필수적이며 모든 오픈 소스 소프트웨어를 사용하는 모든이들의 책임입니다. 
