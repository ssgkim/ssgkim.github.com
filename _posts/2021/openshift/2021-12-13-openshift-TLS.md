---
title: OpenShift v4.x - 인증서 만료 기간
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
description: OpenShift v4.x 버전의 인증서 만료 기간에 대해서 정리 공유합니다.
article_tag1: OpenShift
article_tag2: TLS
article_tag3: infra
article_section: IT근황 공유하기
meta_keywords: openshift,kubernetes,cncf,infra,tls
last_modified_at: '2021-12-13 23:00:00 +0800'
---

OpenShift v4.x 버전의 인증서 만료 기간에 대해서 정리한다.  

## 0. 공통
- GMT(Greenwich Mean Time)+0, Time Zone 기준
- ignition 생성 시점부터 인증서 발급.

## 1. Bootstrap API 인증서
OpenShift 클러스터를 초기 구성을 위해 생성되는 Bootstrap API 서버의 인증서 만료 기간은 24h 동안 유효하다.

    const (
            // ValidityOneDay sets the validity of a cert to 24 hours.
            ValidityOneDay = time.Hour * 24
    )

만료 후 OpenShift 설치를 시도하는 경우 아래와 같이 인증서 만료 메세지가 출력 되고 설치에 실패하므로,  
인증서 만료시 ignition 재생성과 RHCOS 재설치가 필요 하다.

    x509: certificate has expired or not yet valid

**- RefURL**  
[1]: [OpenShift Docs - Bootstrap Certificates](https://docs.openshift.com/container-platform/4.7/security/certificate_types_descriptions/bootstrap-certificates.html)  
[2]: [GitHub - OpenShift Installer: pkg/asset/tls/tls.go#L24-L25](https://github.com/openshift/installer/blob/release-4.7/pkg/asset/tls/tls.go#L24-L25)

## 2. Root CA 인증서
만료 기간은 10y이며, 기간은 클러스터내에서 사용자가 임의로 변경 할 수 없다.

    const (
        // ValidityTenYears sets the validity of a cert to 10 years.
        ValidityTenYears = ValidityOneYear * 10
    )

**- RefURL**  
[3]: [OpenShift Docs - Control Plane Certificates](https://docs.openshift.com/container-platform/4.7/security/certificate_types_descriptions/control-plane-certificates.html)  
[4]: [GitHub - OpenShift Installer: pkg/asset/tls/tls.go#L30-L31](https://github.com/openshift/installer/blob/release-4.7/pkg/asset/tls/tls.go#L30-L31)

## 3. 노드(Kubelet) 인증서
RootCA 인증서를 기반으로 생성되며, 노드가 추가 된 기점으로 부터 1y의 만료 기간을 갖는다.  
OpenShift(kubelet-managed)에서 만료되기 80% 전에 인증서 승인을 자동화 해준다.

    const (
        // ValidityOneYear sets the validity of a cert to 1 year.
        ValidityOneYear = ValidityOneDay * 365
    )

**- RefURL**  
[5]: [OpenShift Docs - Node Certificates](https://docs.openshift.com/container-platform/4.7/security/certificate_types_descriptions/node-certificates.html)  
[6]: [GitHub - OpenShift Installer: pkg/asset/tls/tls.go#L27-L28](https://github.com/openshift/installer/blob/release-4.7/pkg/asset/tls/tls.go#L27-L28)

## 4. ETCD 인증서
Server, Client, Peer 관련 인증서는 기본 3y(26,280h)의 만료 기간을 가진다.  

본 만료 기간 설정 부분은 OpenShift v4.4 버전까지만 아래 bootkube 스크립트에서만 유효하며,  
v4.5 버전 이후 부터는 스크립트에서 설정 부분이 제외 되었고, RootCA를 통해 Server, Client, Peer 인증서를 발급 받는 것으로 변경 되었다.

    # We originally wanted to run the etcd cert signer as
    # a static pod, but kubelet could't remove static pod
    # when API server is not up, so we have to run this as
    # podman container.
    # See https://github.com/kubernetes/kubernetes/issues/43292
    
    echo "Starting etcd certificate signer..."
    
    trap "podman rm --force etcd-signer" ERR
    
    bootkube_podman_run \
    	--name etcd-signer \
    	--detach \
    	--volume /opt/openshift/tls:/opt/openshift/tls:ro,z \
    	"${KUBE_ETCD_SIGNER_SERVER_IMAGE}" \
    	serve \
    	--cacrt=/opt/openshift/tls/etcd-signer.crt \
    	--cakey=/opt/openshift/tls/etcd-signer.key \
    	--metric-cacrt=/opt/openshift/tls/etcd-metric-signer.crt \
    	--metric-cakey=/opt/openshift/tls/etcd-metric-signer.key \
    	--servcrt=/opt/openshift/tls/kube-apiserver-lb-server.crt \
    	--servkey=/opt/openshift/tls/kube-apiserver-lb-server.key \
    	--servcrt=/opt/openshift/tls/kube-apiserver-internal-lb-server.crt \
    	--servkey=/opt/openshift/tls/kube-apiserver-internal-lb-server.key \
    	--servcrt=/opt/openshift/tls/kube-apiserver-localhost-server.crt \
    	--servkey=/opt/openshift/tls/kube-apiserver-localhost-server.key \
    	--address=0.0.0.0:6443 \
    	--insecure-health-check-address=0.0.0.0:6080 \
    	--csrdir=/tmp \
    	--peercertdur=26280h \
    	--servercertdur=26280h \
    	--metriccertdur=26280h

**- RefURL**  
[7]: [OpenShift Docs - ETCD Certificates](https://docs.openshift.com/container-platform/4.7/security/certificate_types_descriptions/etcd-certificates.html)  
[8]: [GitHub - OpenShift Installer: data/data/bootstrap/files/usr/local/bin/bootkube.sh.template#L106-L108](https://github.com/openshift/installer/blob/release-4.4/data/data/bootstrap/files/usr/local/bin/bootkube.sh.template#L106-L108)

## 5. Ingress 인증서
Router Pod의 인증서로써 기본 2y의 만료기간을 갖는다.  
Production 서버에서는 OpenShift 설치시 기본적으로 제공 되는 Ingress 인증서는 권장하지 않으며,  
사용자 정의 인증서를 통해 만료 기간을 늘릴 것을 권장한다.

### 5.1. 계산식

    - NotBefore
    재생성 시점의 시간에서 1/sec를 뺀 시간으로 지정 된다.
    
    - NotAfter
    (2y * 365day) * 24h = 17520h로 지정 된다.

계산식에 의해 생성 된 인증서가 만료 시점 80%(2개월 전)일 경우 Ingress Operator가 자체 서명된 인증서를 재생성 한다.  
재생성시 OpenShift 클러스터 전역의 Service CA가 재발급 되어 클러스터 서비스가 재기동이 되므로,  
Production 환경에서는 사용자 정의 인증서를 사용하여 만료 기간을 길게 설정하여 사용하는 것을 권장 한다.

    // generateRouterCA generates and returns a CA certificate and key.
    func generateRouterCA() ([]byte, []byte, error) {
    	signerName := fmt.Sprintf("%s@%d", "ingress-operator", time.Now().Unix())
    
    	privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    	if err != nil {
    		return nil, nil, fmt.Errorf("failed to generate key: %v", err)
    	}
    
    	root := &x509.Certificate{
    		Subject: pkix.Name{CommonName: signerName},
    
    		SignatureAlgorithm: x509.SHA256WithRSA,
    
    		NotBefore:    time.Now().Add(-1 * time.Second),
    		NotAfter:     time.Now().Add(2 * 365 * 24 * time.Hour),
    		SerialNumber: big.NewInt(1),
    
    		KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
    		BasicConstraintsValid: true,
    
    		IsCA: true,
    
    		// Don't allow the CA to be used to make another CA.
    		MaxPathLen:     0,
    		MaxPathLenZero: true,
    	}

**- RefURL**  
[9]: [OpenShift Docs - Ingress Certificates](https://docs.openshift.com/container-platform/4.7/security/certificate_types_descriptions/ingress-certificates.html)  
[10]: [OpenShift Docs - Service CA certificates](https://docs.openshift.com/container-platform/4.7/security/certificate_types_descriptions/service-ca-certificates.html#services)  
[11]: [GitHub - OpenShift Cluster Ingress Operator: pkg/operator/controller/certificate/ca.go#L61-L87](https://github.com/openshift/cluster-ingress-operator/blob/master/pkg/operator/controller/certificate/ca.go#L61-L87)
