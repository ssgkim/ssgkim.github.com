---
title: openshift 4.10 단일 노드, 설치 후, lvm 및 nfs
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
description: OpenShift SNO 설치 및 lvm nfs 세팅
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-07-16 23:00:00 +0800'
---



# openshift 4.10 단일 노드, 설치 후, lvm 및 nfs

단일 노드 ocp의 경우 별도의 하드 디스크가 있는 경우 lvm 연산자를 사용하여 자동으로 lvm을 할당하고 애플리케이션을 위한 스토리지를 생성할 수 있습니다. 다음으로 이 스토리지를 기반으로 nfs를 구성하고 클러스터 내에서 nfs 서비스가 됩니다.



# lvm Operator 설치

로컬 스토리지가 필요하고 단일 노드 openshift이므로 lvm operator를 사용하고 operator hub에서 operator를 찾고 다음을 설치합니다.

![img](https://wangzheng422.github.io/docker_env/ocp4/4.10/imgs/20220519161647.png)

lvm 연산자는 TP에 있으므로 버그가 있으므로 수정이 필요합니다.

```bash
# oc create ns lvm-operator-system

cat << EOF > /data/install/lvm-operator.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: lvm-operator-system
  annotations:
    workload.openshift.io/allowed: management
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: lvm-operator-system
  namespace: lvm-operator-system
spec:
  targetNamespaces:
  - lvm-operator-system
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: odf-lvm-operator
  namespace: lvm-operator-system
spec:
  channel: "stable-4.10"
  installPlanApproval: Manual
  name: odf-lvm-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
oc create -f /data/install/lvm-operator.yaml

# oc delete -f /data/install/lvm-operator.yaml


ssh -tt core@192.168.7.13 -- lsblk
# NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
# sr0     11:0    1  1024M  0 rom
# vda    252:0    0   120G  0 disk
# ├─vda1 252:1    0     1M  0 part
# ├─vda2 252:2    0   127M  0 part
# ├─vda3 252:3    0   384M  0 part /boot
# └─vda4 252:4    0 119.5G  0 part /sysroot
# vdb    252:16   0   100G  0 disk

oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:lvm-operator-system:topolvm-controller -n lvm-operator-system

oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:lvm-operator-system:vg-manager -n lvm-operator-system

oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:lvm-operator-system:topolvm-node -n lvm-operator-system

cat << EOF > /data/install/lvm.op.yaml
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster-sample
spec:
  storage:
    deviceClasses:
    - name: vg1
    #   thinPoolConfig:
    #     name: thin-pool-1
    #     sizePercent: 50
    #     overprovisionRatio: 50
EOF
oc create -n lvm-operator-system -f /data/install/lvm.op.yaml

kubectl patch storageclass odf-lvm-vg1 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'


cat << EOF > /data/install/lvm.op.pvc.sample.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lvm-file-pvc
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: odf-lvm-vg1
EOF
oc create -f /data/install/lvm.op.pvc.sample.yaml -n default

cat <<EOF > /data/install/lvm.op.app.sample.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-file
spec:
  containers:
  - name: app-file
    image: registry.access.redhat.com/ubi8/ubi:8.4
    imagePullPolicy: IfNotPresent
    command: ["/usr/bin/bash", "-c", "/usr/bin/tail -f /dev/null"]
    volumeMounts:
    - mountPath: "/mnt/file"
      name: lvm-file-pvc
  volumes:
    - name: lvm-file-pvc
      persistentVolumeClaim:
        claimName: lvm-file-pvc
EOF
oc create -f /data/install/lvm.op.app.sample.yaml -n default
```

# 클러스터 내부에 nfs 서비스 설치

- NFS Ganesha 서버 및 외부 프로비저닝 도구
  - [wzh의 포크](https://github.com/wangzheng422/nfs-ganesha-server-and-external-provisioner)

```bash
oc create ns nfs-system

oc project nfs-system

cd /data/install

wget -O nfs.all.yaml https://raw.githubusercontent.com/wangzheng422/nfs-ganesha-server-and-external-provisioner/wzh/deploy/openshift/nfs.all.yaml

oc create -n nfs-system -f nfs.all.yaml

# try it out

wget -O nfs.demo.yaml https://raw.githubusercontent.com/wangzheng422/nfs-ganesha-server-and-external-provisioner/wzh/deploy/openshift/nfs.demo.yaml

oc create -n default -f nfs.demo.yaml
```

{: .notice--info}

**참고자료** <br>
-- [https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators]({{"https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators"}}){:target="_blank"}<br>
{: .notice--info}
