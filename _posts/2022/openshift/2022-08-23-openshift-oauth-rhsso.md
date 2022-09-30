---
title: Openshift는 oauth 인증 RHSSO 사용 법
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
description: SSO Operator 활용 및 사용
article_tag1: OpenShift
article_tag2: Operator
article_tag3: OpenSource
article_section: IT근황 공유하기
meta_keywords: openshift,monitoring,telemetry,runtimes
last_modified_at: '2022-08-23 23:00:00 +0800'
---

# Openshift는 oauth 인증에 rh sso를 사용 법

공식 문서는 잘 작성되었지만 ocp 3.11을 기반으로 하므로 조정해야 할 몇 가지 구성 요소가 있습니다.

- 카탈로그를 통해 배포할 때 관리자 사용자 이름과 암호를 설정해야 합니다.
- 발급자 URL: https://sso-sso-app-demo.apps.ocpef0a.sandbox1717.opentlc.com/auth/realms/OpenShift
- 유효한 리디렉션 URI: https://oauth-openshift.apps.ocpef0a.sandbox1717.opentlc.com/*
- ca.crt 파일은 웹 인터페이스에 업로드할 수 있지만 어떤 파일을 업로드해야 하는지는 openshift-ingress-operator의 router-ca에 있는 tls.crt입니다.
- openid의 로그인 방법이 인터페이스에 항상 표시될 수는 없습니다. 이 경우 시스템 인터페이스로 돌아가서 점프하고 다시 새로고침해야 합니다. 로그인 인터페이스를 항상 새로 고치는 것은 쓸모가 없습니다. 프론트 엔드 페이지의 작은 버그일 것입니다.
- 사용자가 rh sso에서 단일 지점 인증을 받은 후 openshift를 종료하고 다른 사용자로 로그인하려는 경우 허용되지 않습니다. 이 경우 rh sso에 로그인하여 이전 사용자 세션에서 로그아웃한 후 openshift에서 새 사용자로 로그인해야 합니다.
- oauth 아이덴티티 공급자를 추가하는 것은 쉽지만 인터페이스를 제거하지는 않습니다. 이 경우 Identity Providers의 yaml 파일을 직접 변경하고 해당 구성을 삭제하는 것만 가능합니다.

## [자세한 단계]

다음은 구성 프로세스의 화면 녹화입니다.

- https://www.ixigua.com/i6800709743808610827/
- https://youtu.be/Ak9qdgIbOic

프로젝트 sso-app-demo 만들기 ![img](https://wangzheng422.github.io/docker_env/ocp4/4.3/imgs/2020-03-04-19-24-18.png)

카탈로그에서 생성할 sso를 선택하고 나중에 문제를 저장하기 위해 sso 관리자 암호 설정에 주의하십시오. https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.3/html-single/red_hat_single_sign-on_for_openshift/index#deploying_the_red_hat_single_sign_on_image_using_the_application_template ![img](https://wangzheng422.github.io/docker_env/ocp4/4.3/imgs/2020-03-04-19-25-18.png)

그런 다음 rh sso에 로그인하고 공식 문서 https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.3/html-single/red_hat_single_sign-on_for_openshift/index#OSE-SSO-에 따라 구성합니다. AUTH-TUTE

## [대체 명령]

```bash
# oc -n openshift import-image redhat-sso73-openshift:1.0

# oc new-project sso-app-demo
# oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default

# oc policy remove-role-from-user view system:serviceaccount:$(oc project -q):default

# get issuer url
curl -k https://sso-sso-app-demo.apps.ocpef0a.sandbox1717.opentlc.com/auth/realms/OpenShift/.well-known/openid-configuration | python -m json.tool | grep issuer

# curl -k https://sso-sso-app-demo.apps.ocpef0a.sandbox1717.opentlc.com/auth/realms/OpenShift/.well-known/openid-configuration | jq | less

# # on mac create a ca
# cd ~/Downloads/tmp/tmp/
# openssl req \
#    -newkey rsa:2048 -nodes -keyout redhat.ren.key \
#    -x509 -days 3650 -out redhat.ren.crt -subj \
#    "/C=CN/ST=GD/L=SZ/O=Global Security/OU=IT Department/CN=*.redhat.ren"
# # upload crt to ocp
# oc create configmap ca-config-map --from-file=ca.crt=./redhat.ren.crt -n openshift-config

# oc delete configmap ca-config-map -n openshift-config

oc get secrets router-ca -n openshift-ingress-operator -o jsonpath='{.data.tls\.crt}' | base64 -d > router.ca.crt

# oc get secrets router-ca -n openshift-ingress-operator -o jsonpath='{.data.tls\.key}' | base64 -d

# oc get OAuthClient

# if you want to debug, https://bugzilla.redhat.com/show_bug.cgi?id=1744599
oc patch authentication.operator cluster --type=merge -p "{\"spec\":{\"operatorLogLevel\": \"TraceAll\"}}"
oc patch authentication.operator cluster --type=merge -p "{\"spec\":{\"operatorLogLevel\": \"\"}}"

# update imate stream for offline
oc patch -n openshift is mysql -p "{\"spec\":{\"tags\":[{\"name\": \"5.7\",\"from\":{\"name\":\"registry.redhat.ren:5443/registry.redhat.io/rhscl/mysql-57-rhel7:latest\"}}]}}"
oc patch -n openshift is mysql -p "{\"spec\":{\"tags\":[{\"name\": \"8.0\",\"from\":{\"name\":\"registry.redhat.ren:5443/registry.redhat.io/rhscl/mysql-80-rhel7:latest\"}}]}}"
oc patch -n openshift is redhat-sso73-openshift -p "{\"spec\":{\"tags\":[{\"name\": \"1.0\",\"from\":{\"name\":\"registry.redhat.ren:5443/registry.redhat.io/redhat-sso-7/sso73-openshift:1.0\"}}]}}"
oc patch -n openshift is redhat-sso73-openshift -p "{\"spec\":{\"tags\":[{\"name\": \"latest\",\"from\":{\"name\":\"registry.redhat.ren:5443/registry.redhat.io/redhat-sso-7/sso73-openshift:1.0\"}}]}}"

oc create is ipa-server -n openshift
```


{: .notice--info}

**참고자료** <br>
-- [https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators]({{"https://www.redhat.com/ko/technologies/cloud-computing/openshift/what-are-openshift-operators"}}){:target="_blank"}<br>
{: .notice--info}
