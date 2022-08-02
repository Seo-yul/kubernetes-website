---
##
##
##
##
title: 프로젝티드 볼륨
content_type: concept
weight: 21 # just after persistent volumes
---

<!-- overview -->

이 페이지에서는 쿠버네티스의 _프로젝티드 볼륨_  에 대해 설명한다. [볼륨](/ko/docs/concepts/storage/volumes/)에 대해 익숙해지는 것을 추천한다.

<!-- body -->

## Introduction

`프로젝티드볼륨`은 여러 기존 볼륨 소스(sources)를 동일한 디렉토리에 매핑한다.

현재, 아래와 같은 볼륨 유형 소스를 프로젝티드(projected)할 수 있다.

* [`시크릿(secret)`](/ko/docs/concepts/storage/volumes/#secret)
* [`downwardAPI`](/ko/docs/concepts/storage/volumes/#downwardapi)
* [`컨피그맵(configMap)`](/ko/docs/concepts/storage/volumes/#configmap)
* [`서비스어카운트토큰(serviceAccountToken)`](#serviceaccounttoken)

모든 소스는 파드와 같은 네임스페이스에 있어야 한다.
자세한 내용은 [all-in-one volume](https://git.k8s.io/design-proposals-archive/node/all-in-one-volume.md) 문서를 참고한다.

### 시크릿, downwardAPI, 컨피그맵 구성 예시 {#시크릿-downwardapi-컨피그맵-구성-예시}

{{< codenew file="pods/storage/projected-secret-downwardapi-configmap.yaml" >}}

### 기본 권한이 아닌 모드 설정의 시크릿 구성 예시 {#비기본-권한-모드설정-시크릿-구성-예시}

{{< codenew file="pods/storage/projected-secrets-nondefault-permission-mode.yaml" >}}

각 프로젝티드볼륨 소스는 `sources` 아래의 스펙에 나열된다.
파라미터는 두 가지 예외를 제외하고 동일하다.

* 시크릿의 경우 컨피그맵 명명법과 동일하도록
  `secretName` 필드가 `name`으로 변경되었다.
* `defaultMode`의 경우 볼륨 소스별 각각 명시할 수 없고
  프로젝티드 수준에서만 명시할 수 있다. 그러나 위의 그림처럼 각 개별 프로젝션에 대해
  `mode`를 명시적으로 지정할 수 있다.

## 서비스어카운트토큰 프로젝티드볼륨 {#서비스어카운트토큰-프로젝티드볼륨}
`TokenRequestProjection` 기능이 활성화 된 경우
파드의 지정된 경로에 [서비스어카운트토큰](/docs/reference/access-authn-authz/authentication/#service-account-tokens)을
주입할 수 있다. 

{{< codenew file="pods/storage/projected-service-account-token.yaml" >}}

위 예시에는 파드에 주입된 서비스어카운트토큰이 포함된 프로젝티드볼륨이 있다. 
이 파드의 컨테이너는 쿠버네티스 API 서버에 접근하여 서비스어카운트토큰을 사용해
파드의 [서비스어카운트](/docs/tasks/configure-pod-container/configure-service-account/)로 인증할 수 있다.
`audience` 필드는 토큰의 의도된 대상을 포함한다.
토큰 수신자는 토큰의 대상으로 지정된 식별자로 자신을 식별해야 하며,
그렇지 않으면 토큰을 거부해야 한다.
이 필드는 선택 사항으로 API 서버의 식별자로 기본 설정된다.

`expirationSeconds`는 서비스어카운트토큰의 예상 유효 기간이다.
기본값으로 1시간이며 최소 10분 (600초) 이상이어야 한다.
관리자는 API 서버 옵션 `--service-account-max-token-expiration`으로
값을 지정하여 최대값을 제한할 수 있다.
`path` 필드는 프로젝티드볼륨의 마운트 지점에 대한 상대 경로를 지정한다.

{{< note >}}
[`하위 경로`](/docs/concepts/storage/volumes/#using-subpath) 볼륨 마운트로 프로젝티드볼륨 소스를 사용하는 컨테이너는
해당 볼륨 소스에 대한 업데이트를 수신하지 않는다.
{{< /note >}}

## 보안 컨텍스트 상호작용 {#보안-컨텍스트-상호작용}

프로젝티드 서비스어카운트 볼륨 강화에서 소개한 파일 권한 처리를 위한 [제안](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/2451-service-account-token-volumes#proposal)은 올바른 소유자 권한 세트를 가진 프로젝티드파일을 도입했다.

### 리눅스 {#리눅스}

프로젝티드볼륨과 파드
[`보안 컨텍스트`](/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context)에
`RunAsUser`가 설정된 리눅스 파드에서는
프로젝티드파일이 컨테이너 사용자 소유권을 포함한 올바른 소유권 집합을 가진다.

### 윈도우 {#윈도우}

In Windows pods that have a projected volume and `RunAsUsername` set in the
Pod `SecurityContext`, the ownership is not enforced due to the way user
accounts are managed in Windows. Windows stores and manages local user and group
accounts in a database file called Security Account Manager (SAM). Each
container maintains its own instance of the SAM database, to which the host has
no visibility into while the container is running. Windows containers are
designed to run the user mode portion of the OS in isolation from the host,
hence the maintenance of a virtual SAM database. As a result, the kubelet running
on the host does not have the ability to dynamically configure host file
ownership for virtualized container accounts. It is recommended that if files on
the host machine are to be shared with the container then they should be placed
into their own volume mount outside of `C:\`.

기본적으로, 프로젝티드파일은 예제의 프로젝티드볼륨파일처럼
아래와 같은 소유권을 가진다.
```powershell
PS C:\> Get-Acl C:\var\run\secrets\kubernetes.io\serviceaccount\..2021_08_31_22_22_18.318230061\ca.crt | Format-List

Path   : Microsoft.PowerShell.Core\FileSystem::C:\var\run\secrets\kubernetes.io\serviceaccount\..2021_08_31_22_22_18.318230061\ca.crt
Owner  : BUILTIN\Administrators
Group  : NT AUTHORITY\SYSTEM
Access : NT AUTHORITY\SYSTEM Allow  FullControl
         BUILTIN\Administrators Allow  FullControl
         BUILTIN\Users Allow  ReadAndExecute, Synchronize
Audit  :
Sddl   : O:BAG:SYD:AI(A;ID;FA;;;SY)(A;ID;FA;;;BA)(A;ID;0x1200a9;;;BU)
```
This implies all administrator users like `ContainerAdministrator` will have
read, write and execute access while, non-administrator users will have read and
execute access.

{{< note >}}
In general, granting the container access to the host is discouraged as it can
open the door for potential security exploits.

Creating a Windows Pod with `RunAsUser` in it's `SecurityContext` will result in
the Pod being stuck at `ContainerCreating` forever. So it is advised to not use
the Linux only `RunAsUser` option with Windows Pods.
{{< /note >}}
