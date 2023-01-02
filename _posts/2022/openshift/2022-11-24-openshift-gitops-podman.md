---
title: GitOps와 Podman을 사용한 컨테이너 수명 주기 구성
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
description: Podman GitOps 구성
article_tag1: OpenShift
article_tag2: gitops
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-11-23 23:00:00 +0800'
---

# GitOps와 Podman을 사용한 컨테이너 수명 주기 구성



GitOps는 모든 개발자가 익숙하지는 않지만 많은 개발자가 익숙한 도구인 Git을 기반으로 하므로 Kubernetes 애플리케이션의 지속적인 제공을 위한 훌륭한 솔루션입니다. 개발자는 원하는 상태를 Git에 저장하여 배포된 애플리케이션을 관리하고 수정 기록, 코드 검토 및 배포 분기와 같은 이점을 얻을 수 있습니다.

ArgoCD 및 RHACM(Red Hat Advanced Cluster Management)과 같은 GitOps 도구는 대부분 Kubernetes 환경에 중점을 두었지만 Kubernetes가 필요하지 않은 더 가벼운 환경은 어떨까요?

미리 언급하자면! 솔루션이 있습니다. [FetchIt](https://github.com/containers/fetchit) ! Podman을 통해 컨테이너를 배포하고 관리하기 위한 가볍고 간단한 수동 GitOps 도구입니다. FetchIt은 systemd, Ansible 및 podman-kube와 같은 다양한 배포 방법을 지원합니다. 조금 자동화 관점이 많이 포함되고 더 무거운 Kubernetes 클러스터를 배포하는 데 사용하는 것과 동일한 파일을 사용합니다. FetchIt 실행 방법과 FetchIt 작동 방식에 대해 설명합니다. 또한 FetchIt이 지원하는 배포 방법에 대해서도 설명합니다.



## FetchIt 설정

FetchIt에는 구성 파일, Podman 및 적절한 Podman 소켓 실행과 같은 몇 가지 요구 사항이 있습니다. Podman에는 사용자 소켓과 루트 소켓이 있습니다. 사용자 소켓을 활성화 합니다.

`systemctl --user enable podman.socket --now`

루트 사용자의 경우 다음 명령을 사용합니다.

`sudo systemctl enable podman.socket --now`

### FetchIt은 어떻게 작동합니까?

FetchIt은 구성 파일에 지정된 원격 Git 리포지토리를 모니터링하고 자주 폴링하여 변경 사항을 확인하는 방식으로 작동합니다. 대상 리포지토리에 할당된 방법에 따라 변경 사항이 감지되면 FetchIt은 서버의 배포에 필요한 업데이트를 수행합니다. Podman 소켓과 통신하여 컨테이너 또는 포드를 생성하거나 systemd 서비스 파일을 서버에 작성한 다음 실행하면 됩니다.

### FetchIt 실행

FetchIt을 실행하는 두 가지 권장 방법이 있습니다: 시스템 서비스로 또는 컨테이너 내에서 직접. 두 경우 모두 컨테이너 내에서 실행됩니다.

**시스템 서비스로 FetchIt**
실행 Podman 루트 소켓을 사용하여 루트로 FetchIt을 실행하거나 Podman의 사용자 소켓을 사용하여 사용자로 실행할 수 있습니다. 사용하려는 소켓이 활성화되어 있는지 확인하십시오. systemd 서비스는 사용자를 위해 FetchIt 컨테이너를 실행하고 사용자가 해야 할 일은 에 구성을 작성했는지 확인하는 것뿐입니다 `~/.fetchit/config.yaml`.

**컨테이너에서 FetchIt 실행**
다른 방법으로 컨테이너 내에서 직접 실행하려면 몇 가지 옵션과 함께 해당 컨테이너를 실행해야 합니다.

명령은 아래에서 찾을 수 있습니다.

```bash
podman run -d --rm --name fetchit \
  -v fetchit-volume:/opt \
  -v $HOME/.fetchit:/opt/mount \
  -v /run/user/$(id -u)/podman/podman.sock:/run/podman/podman.sock \
  quay.io/fetchit/fetchit-amd:latest
```



*참고:*`--security-opt label=disable` 적용 모드에서 SELinux와 함께 실행 중인 경우 실행 명령에 podman 플래그를 추가하십시오 . 이것은 fetchit 팟(Pod)에서 레이블 분리를 끕니다.

이 명령의 중요한 인수는 볼륨 마운트입니다. 나머지는 선택 사항입니다. `/opt`이 명령 은 FetchIt이 모니터링 중인 Git 리포지토리를 복제하는 의 구성에 지정된 동일한 이름으로 볼륨을 마운트합니다 . 이 명령은 또한 사용자의 config.yaml을 에 마운트합니다 `/opt/mount`. 최종 마운트는 호스트의 podman 소켓을 컨테이너의 해당 소켓에 마운트하는 것입니다. 이제 FetchIt 인스턴스가 컴퓨터에서 실행됩니다.

### FetchIt 구성

YAML 구성 파일은 사용자가 FetchIt을 구성하기 위한 인터페이스입니다. 여기에는 대상 리포지토리 목록, 리포지토리 옵션 및 각 리포지토리에 대해 실행할 메서드가 포함됩니다. 에 마운트되는 Podman 볼륨의 이름을 지정하는 필드도 있습니다 `/opt`. FetchIt GitHub에는 예제로 가득 찬 디렉토리가 있으므로 CI의 모든 메서드를 테스트하는 데 사용되는 구성 파일인 full-suite.yaml을 살펴보겠습니다.

```yaml
targetConfigs:
- url: http://github.com/containers/fetchit
  raw:
  - name: raw-ex
    targetPath: examples/raw
    schedule: "*/1 * * * *"
    pullImage: false
  systemd:
  - name: sysd-ex
    targetPath: examples/systemd
    root: true
    enable: false
    schedule: "*/1 * * * *"
  ansible:
  - name: ans-ex
    targetPath: examples/ansible
    sshDirectory: /root/.ssh
    schedule: "*/1 * * * *"
  filetransfer:
  - name: ft-ex
    targetPath: examples/filetransfer
    destinationDirectory: /tmp/ft
    schedule: "*/1 * * * *"
  branch: main
prune:
  All: true
  Volumes: false
  schedule: "*/1 * * * *"
```



대상 목록을 구성하여 시작합니다. 각 대상에는 URL 필드와 메소드 필드가 있습니다. URL은 대상으로 지정하려는 원격 리포지토리를 가리킵니다. 메서드에는 이 리포지토리에서 실행하려는 각 메서드와 해당 옵션을 정의하는 필드가 포함되어 있습니다. 각 메서드에는 이름 필드, 대상 경로 필드 및 일정 필드가 있습니다. 대상 경로 필드는 해당 메서드가 배포해야 하는 항목을 설명하는 파일이 포함된 디렉터리를 정의합니다. 일정 필드는 메서드가 실행되어야 하는 시기를 cron 형식으로 설명합니다. 메서드의 다른 필드는 일반적으로 해당 메서드에만 적용됩니다. 아래에서 개별 방법을 설명합니다.

**Fetchit 신뢰성**
FetchIt은 가능한 모든 방법을 실행하려고 시도하며 하나가 실패하더라도 다른 방법을 실행합니다. 목표는 일부 오류가 발생하더라도 FetchIt을 계속 실행하고 사용자가 원인을 식별할 수 있도록 이러한 오류가 발생했음을 사용자에게 알리는 것입니다. 이것은 FetchIt에 설계된 안정성 수준입니다.

계속하기 전에 요약하면 다음과 같습니다.

- FetchIt을 사용하려면 사용자가 구성 파일, Podman, 실행 중인 적절한 Podman 소켓 및 허용 모드의 SELinux를 가지고 있어야 합니다.
- FetchIt을 실행하려면 두 가지 옵션이 있습니다. [이 리포지토리](https://github.com/containers/fetchit/tree/main/systemd) 의 두 서비스 파일 중 하나를 사용하는 systemd 서비스 또는 위에서 설명한 대로 컨테이너에서 직접 구성할 수 있음.
- FetchIt의 구성은 대상과 각 대상에 대해 실행할 메서드로 구성됩니다.
- 각 방법에는 실행해야 하는 간격을 지정하는 일정이 필요합니다.
- 많은 방법에는 해당 방법을 사용하여 배포할 파일을 찾을 위치를 지정하기 위해 targetPath 필드가 필요합니다.

## 행동 양식

FetchIt이 git 리포지토리를 나타내는 대상으로 구성되어 있음을 알 수 있습니다. 사용자는 여러 대상을 지정할 수 있으며 각 대상은 아래에 설명된 각 고유 방법 중 하나까지 실행됩니다. 일정은 해당 방법으로 범위가 지정되며 동일한 대상에 대한 다른 방법의 일정은 완전히 독립적입니다. 대상은 실행 중인 동일한 메소드의 두 인스턴스를 가질 수 없습니다. 방법이 컴퓨터의 동일한 디렉터리에 파일을 복사하거나 동일한 이름의 컨테이너를 만드는 경우 사용자가 해결해야 하는 충돌이 발생합니다.

### Raw

원시 방법이 가장 간단합니다. 이를 통해 사용자는 작은 사양에서 컨테이너를 만들 수 있습니다. 다음은 원시 메서드 구성의 예입니다.

```
raw:
  - name: raw-ex
    targetPath: examples/raw
    schedule: "*/1 * * * *"
    pullImage: false
```



이 메서드의 유일한 고유 옵션은 메서드를 실행할 때마다 아래 사양에 지정된 이미지를 강제로 가져오는 pullImage 옵션입니다. targetPath 옵션에 지정된 `examples/raw`디렉터리에서 다음 형식의 YAML 파일을 찾을 수 있습니다.

```yaml
Image: "docker.io/mmumshad/simple-webapp-color:latest"
Name: "colors2"
Env:
  APP_COLOR: "pink"
  tree: "trunk"
Ports:
  - container_port: 8080
    host_port: 9080
    range: 0
```



이 YAML은 컨테이너 이름, 이미지, 환경 변수 및 포트 매핑 목록으로 간단한 컨테이너 구성을 지정합니다. 원시 메서드를 실행할 때마다 FetchIt은 대상 디렉터리의 파일을 구문 분석하고 해당 파일의 이미지가 시스템에 있는지 확인하고 그렇지 않은 경우(또는 pullmage 옵션이 true로 설정된 경우) 이미지를 가져옵니다. 그런 다음 파일에 지정된 이름과 동일한 이름을 가진 이전 컨테이너를 제거하고 파일에 제공된 사양에 따라 새 컨테이너를 만듭니다.

### 체계적인

systemd 메서드를 사용하면 사용자가 적절한 systemd 디렉토리의 로컬 시스템에 저장할 systemd 서비스 파일을 지정할 수 있습니다. systemd 메서드 구성에는 많은 옵션이 있습니다. 첫 번째는 간단한 것입니다.

```yaml
systemd:
  - name: sysd-ex
    targetPath: examples/systemd
    root: true
    enable: false
    schedule: "*/1 * * * *"
```



위의 예에서 볼 수 있듯이 systemd 방법에는 세 가지 고유한 옵션이 있습니다. 첫 번째는 루트 옵션으로, 서비스 파일을 루트 서비스로 실행하기 위해 루트 systemd 디렉토리에 보관할지 또는 루트가 아닌 systemd 디렉토리에 배치하여 루트가 아닌 서비스로 실행할지 여부를 사용자가 결정할 수 있습니다. . 현재 사용자가 FetchIt이 서비스를 자동으로 활성화하기를 원하면 true로 설정해야 합니다. 루트 옵션이 설정되지 않은 경우 사용자는 와 같은 명령을 사용하여 서비스를 직접 활성화해야 합니다 `systemctl –user enable`.

두 번째 옵션은 입니다 . 이 옵션 `enable`을 설정하면 시스템의 적절한 위치에 파일을 저장한 후 자동으로 사용자에 대한 서비스를 활성화합니다. 마지막 옵션은 `restart`systemd가 변경 사항을 감지하면 (변경된 서비스 파일을 적용한 후) 서비스를 다시 시작하는 입니다.

### 파일 전송

파일 전송 방법을 사용하면 사용자가 원격 저장소에서 로컬 시스템의 특정 위치로 파일을 저장할 수 있습니다. 이 방법은 서비스 파일을 올바른 위치로 가져오기 위해 systemd 방법 내에서 사용됩니다. 다음은 구성 예입니다.

```yaml
filetransfer:
  - name: ft-ex
    targetPath: examples/filetransfer
    destinationDirectory: /tmp/ft
    schedule: "*/1 * * * *"
```



이 구성의 경우 고유한 옵션은 `destinationDirectory `옵션입니다. 이 옵션은 파일을 보관할 디렉토리의 이름을 지정합니다.

### 앤서블

Ansible 방법을 사용하면 사용자가 FetchIt을 실행하는 호스트를 수정하기 위해 FetchIt이 배포하는 컨테이너 내에서 대상 경로에 있는 Ansible 플레이북을 실행할 수 있습니다. 다음은 Ansible 방법의 구성 예입니다.

```yaml
 ansible:
  - name: ans-ex
    targetPath: examples/ansible
    sshDirectory: /root/.ssh
    schedule: "*/1 * * * *"
```



여기서 고유한 옵션은 `sshDirectory`. `authorized_keys`이 옵션을 사용하면 FetchIt이 SSH 키와 파일 을 포함하는 필수 .ssh 디렉토리를 마운트할 디렉토리를 알 수 있습니다 .

### Kube

kube 방법은 `podman-play-kube`기능을 사용하여 사용자가 Podman을 통해 실행할 구성 맵, PVC, 포드 및 배포를 정의하는 Kubernetes YAML 사양을 배포할 수 있도록 합니다. PVC는 Podman 볼륨에 매핑되고 구성 맵은 kube 사양이 적용될 때 메모리에 로드됩니다. 다음은 kube 메서드 구성의 예입니다.

```yaml
kube:
  - name: kube-ex
    targetPath: examples/kube
    schedule: "*/1 * * * *"
```



이 방법에 대한 고유한 옵션은 없지만 `targetPath `디렉터리 내에서 다음과 같은 Kubernetes 사양 파일을 찾을 수 있습니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env
data:
  APP_COLOR: blue
  tree: trunk
---
apiVersion: v1
kind: Pod
metadata:
  name: colors_pod
spec:
  containers:
  - name: colors-kubeplay
    image: docker.io/mmumshad/simple-webapp-color:latest
    ports:
    - containerPort: 8080
      hostPort: 7080
    envFrom:
    - configMapRef:
        name: env
        optional: false
```



FetchIt은 `podman play kube`이 파일에서 실행되며 Podman은 파일에서 선언된 포트 매핑과 함께 여기에서 선언된 컨테이너가 있는 파드를 생성하여 파일 시작 부분에서 선언된 구성 맵을 사용합니다. [Podman 설명서](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html)`podman kube play` 에서 자세한 내용 을 확인할 수 있습니다 .

### Prune

prune 옵션은 파일을 사용하지 않아 대상이 필요하지 않은 특별한 방법이지만 사용하지 않는 컨테이너와 이미지를 제거하는 방법으로 사용됩니다. `podman system prune`구성에 제공된 일정에 따라 분석법 구성에 설정된 옵션으로 실행됩니다 .

```yaml
prune:
  All: true
  Volumes: false
  schedule: "*/1 * * * *"
```



`podman system prune`구성의 All 및 Volumes 옵션 은 동일한 이름을 공유하는 플래그에 매핑됩니다 . 자세한 내용 `podman system prune`은 [설명서](https://docs.podman.io/en/latest/markdown/podman-system-prune.1.html) 를 참조하십시오 .

### 구성 새로고침

마지막으로 구성 다시 로드 방법은 깨끗하고 체계적인 자동 업데이트와 같은 특별한 방법이기도 합니다. 이 방법은 약간 복잡하고 자체 기사가 필요하지만 완전성을 위해 여기에서 언급합니다.

## 결론

이러한 모든 방법이 GitOps 방식으로 사용된다는 점을 다시 언급하는 것이 중요합니다. FetchIt이 변경 사항을 감시하는 구성 파일에서 알고 있는 원격 대상 리포지토리가 있습니다. 각 변경 사항에 대해 FetchIt은 해당 파일에 대한 업데이트를 가져오고 저장소에서 선언된 내용과 일치하도록 실행 중인 컨테이너의 상태를 업데이트하기 위해 실행해야 하는 항목을 실행합니다. 이로 인해 FetchIt은 각 시스템이 원격 저장소 내에서 선언된 상태와 일치하도록 배포를 최소한으로 변경하도록 합니다.

[FetchIt GitHub 리포지토리](https://github.com/containers/fetchit) 에는 FetchIt 을 구성하는 방법에 대해 자세히 알아보려는 경우 탐색할 수 있는 매우 완전한 [예제 디렉터리 가 있습니다. ](https://github.com/containers/fetchit/tree/main/examples)

프로젝트에 기여하는 데 관심이 있으시면 언제든지 [저장소](https://github.com/containers/fetchit) 를 방문하여 공개 이슈 또는 풀 요청을 하십시오.
