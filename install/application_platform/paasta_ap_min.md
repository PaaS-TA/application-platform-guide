### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > PaaS-TA AP - min

## Table of Contents

1. [개요](#1)  
 1.1. [목적](#1.1)  
 1.2. [범위](#1.2)  
 1.3. [참고 자료](#1.3)  
2. [PaaS-TA AP min 설치](#2)  
 2.1. [Prerequisite](#2.1)  
 2.2. [설치 파일 다운로드](#2.2)  
 2.3. [Stemcell 업로드](#2.3)  
 2.4. [Runtime Config 설정](#2.4)  
 2.5. [Cloud Config 설정](#2.5)  
 2.6. [PaaS-TA AP min 설치 파일](#2.6)  
　2.6.1. [PaaS-TA AP min 설치 Variable 파일](#2.6.1)    
　2.6.2. [PaaS-TA AP min Operation 파일](#2.6.2)  
　2.6.3. [PaaS-TA AP min 설치 Shell Scripts](#2.6.3)  
 2.7. [PaaS-TA AP min 설치](#2.7)  
 2.8. [PaaS-TA AP min 로그인](#2.8)   

# <div id='1'/>1.  문서 개요

## <div id='1.1'/>1.1. 목적
본 문서는 Monitoring을 적용하지 않은 PaaS-TA Application Platform 경량화(이하 PaaS-TA AP min)을 수동으로 설치하기 위한 가이드를 제공하는 데 그 목적이 있다.

<br>

## <div id='1.2'/>1.2. 범위
PaaS-TA AP min은 bosh-deployment를 기반으로 한 BOSH 환경에서 설치하며 paasta-deployment v5.8.5-min의 설치를 기준으로 가이드를 작성하였다.  
PaaS-TA AP min은 VMware vSphere, Google Cloud Platform, Amazon Web Services EC2, OpenStack, Microsoft Azure 등의 IaaS를 지원하며,  paasta-deployment v5.8.5-min에서 검증한 IaaS 환경은 AWS, OpenStack, vSphere, Azure, GCP 환경이다.

<br>

## <div id='1.3'/>1.3. 참고 자료

본 문서는 Cloud Foundry의 BOSH Document와 Cloud Foundry Document를 참고로 작성하였다.

BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  
BOSH Deployment: [https://github.com/cloudfoundry/bosh-deployment](https://github.com/cloudfoundry/bosh-deployment)  
CF Deployment: [https://github.com/cloudfoundry/cf-deployment](https://github.com/cloudfoundry/cf-deployment)  

<br><br>

# <div id='2'/>2. PaaS-TA AP min 설치
## <div id='2.1'/>2.1. Prerequisite

- BOSH2 기반의 BOSH를 설치한다.
- PaaS-TA AP min 설치는 BOSH를 설치한 Inception(설치 환경)에서 작업한다.
- PaaS-TA AP min 설치를 위해 BOSH LOGIN을 진행한다.
- 가이드 내에 BOSH 폴더에서 실행되는 작업은 BOSH 설치 가이드에서 다운받은 paasta-deployment를 이용한다.

<br>

## <div id='2.2'/>2.2. 설치 파일 다운로드
- PaaS-TA AP min를 설치하기 위한 deployment가 존재하지 않는다면 다운로드 받는다

```
$ mkdir -p ~/workspace
$ cd ~/workspace
$ git clone https://github.com/PaaS-TA/common.git
$ cd ~/workspace
$ git clone https://github.com/PaaS-TA/paasta-deployment.git -b v5.8.5-min paasta-deployment-min
```

<br>

## <div id='2.3'/>2.3. Stemcell 업로드
Stemcell은 배포 시 생성되는 VM Base OS Image이다.  
paasta-deployment v5.8.5-min은 Ubuntu bionic stemcell 1.171을 기반으로 한다.  
기본적인 Stemcell 업로드 명령어는 다음과 같다.  
```                     
$ bosh -e ${BOSH_ENVIRONMENT} upload-stemcell {URL}
```

paasta-deployment는 v5.5.0 부터 Stemcell 업로드 스크립트를 지원하며, BOSH 로그인 후 다음 명령어를 수행하여 Stemcell을 올린다.  
BOSH_ENVIRONMENT는 BOSH 설치 시 사용한 Director 명이고, CURRENT_IAAS는 배포된 환경 IaaS(aws, azure, gcp, openstack, vsphere, 그외 입력시 bosh-lite)에 맞게 입력을 한다.
<br>(paasta-deployment에서 제공되는 create-bosh-login.sh을 이용하여 BOSH LOGIN시 BOSH_ENVIRONMENT와 CURRENT_IAAS는 자동입력된다.)

- Stemcell 업로드 Script의 설정 수정 (BOSH_ENVIRONMENT 수정)

> $ vi ~/workspace/paasta-deployment/bosh/upload-stemcell.sh
```                     
#!/bin/bash
STEMCELL_VERSION=1.171
CURRENT_IAAS="${CURRENT_IAAS}"				# IaaS Information (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 aws/azure/gcp/openstack/vsphere/bosh-lite 입력)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"			# bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

if [[ ${CURRENT_IAAS} = "aws" ]]; then
        bosh -e ${BOSH_ENVIRONMENT} upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/${STEMCELL_VERSION}/bosh-stemcell-${STEMCELL_VERSION}-aws-xen-hvm-ubuntu-bionic-go_agent.tgz -n
elif [[ ${CURRENT_IAAS} = "azure" ]]; then
        bosh -e ${BOSH_ENVIRONMENT} upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/${STEMCELL_VERSION}/bosh-stemcell-${STEMCELL_VERSION}-azure-hyperv-ubuntu-bionic-go_agent.tgz -n
elif [[ ${CURRENT_IAAS} = "gcp" ]]; then
        bosh -e ${BOSH_ENVIRONMENT} upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/${STEMCELL_VERSION}/bosh-stemcell-${STEMCELL_VERSION}-google-kvm-ubuntu-bionic-go_agent.tgz -n
elif [[ ${CURRENT_IAAS} = "openstack" ]]; then
        bosh -e ${BOSH_ENVIRONMENT} upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/${STEMCELL_VERSION}/bosh-stemcell-${STEMCELL_VERSION}-openstack-kvm-ubuntu-bionic-go_agent.tgz -n
elif [[ ${CURRENT_IAAS} = "vsphere" ]]; then
        bosh -e ${BOSH_ENVIRONMENT} upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/${STEMCELL_VERSION}/bosh-stemcell-${STEMCELL_VERSION}-vsphere-esxi-ubuntu-bionic-go_agent.tgz -n
elif [[ ${CURRENT_IAAS} = "bosh-lite" ]]; then
        bosh -e ${BOSH_ENVIRONMENT} upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/${STEMCELL_VERSION}/bosh-stemcell-${STEMCELL_VERSION}-warden-boshlite-ubuntu-bionic-go_agent.tgz -n
else
        echo "plz check CURRENT_IAAS"
fi

```

- Stemcell 업로드 Script 실행

```
$ cd ~/workspace/paasta-deployment/bosh
$ source upload-stemcell.sh
```

<br>

## <div id='2.4'/>2.4. Runtime Config 설정
Runtime config는 BOSH로 배포되는 VM에 일괄 적용되는 설정이다.
기본적인 Runtime Config 설정 명령어는 다음과 같다.  
```                     
$ bosh -e ${BOSH_ENVIRONMENT} update-runtime-config {PATH} --name={NAME}
```

PaaS-TA AP min에서 적용하는 Runtime Config는 다음과 같다.  

- DNS Runtime Config  
  PaaS-TA AP 4.0부터 적용되는 부분으로 PaaS-TA Component에서 Consul이 대체된 Component이다.  
  PaaS-TA AP Component 간의 통신을 위해 BOSH DNS 배포가 선행되어야 한다.  

- OS Configuration Runtime Config  
  BOSH Linux OS 구성 릴리스를 이용하여 sysctl을 구성한다.  

paasta-deployment는 v5.5.0 부터 Runtime Config 설정 스크립트를 지원하며, BOSH 로그인 후 다음 명령어를 수행하여 Runtime Config를 설정한다.  

  - Runtime Config 업데이트 Script 수정 (BOSH_ENVIRONMENT 수정)
> $ vi ~/workspace/paasta-deployment/bosh/update-runtime-config.sh
```                     
#!/bin/bash

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"			 # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} update-runtime-config -n runtime-configs/dns.yml
bosh -e ${BOSH_ENVIRONMENT} update-runtime-config -n --name=os-conf runtime-configs/os-conf.yml
```
- Runtime Config 업데이트 Script 실행
```                     
$ cd ~/workspace/paasta-deployment/bosh
$ source update-runtime-config.sh
```

  - Runtime Config 확인  
  ```  
  $ bosh -e ${BOSH_ENVIRONMENT} runtime-config
  $ bosh -e ${BOSH_ENVIRONMENT} runtime-config --name=os-conf
  ```

<br>

## <div id='2.5'/>2.5. Cloud Config 설정

BOSH를 통해 VM을 배포 시 IaaS 관련 Network, Storage, VM 관련 설정을 Cloud Config로 정의한다.  
deployment 설치 파일을 내려받으면 ~/workspace/paasta-deployment-min/cloud-config 디렉터리 이하에 IaaS 별 Cloud Config 예제를 확인할 수 있으며, 예제를 참고하여 cloud-config.yml을 IaaS에 맞게 수정한다.  
PaaS-TA AP min 배포 전에 Cloud Config를 BOSH에 적용해야 한다.

- AWS을 기준으로 한 [cloud-config.yml](https://github.com/PaaS-TA/paasta-deployment/blob/master/cloud-config/aws-cloud-config.yml) 예제

```
## azs :: 가용 영역(Availability Zone)을 정의한다.
azs:
- cloud_properties:
    availability_zone: ap-northeast-2a
  name: z1
- cloud_properties:
    availability_zone: ap-northeast-2a
  name: z2

... ((생략)) ...

## compilation :: 컴파일 가상머신이 생성될 가용 영역 및 가상머신 유형 등을 정의한다.
compilation:
  az: z4
  network: default
  reuse_compilation_vms: true
  vm_type: xlarge
  workers: 5

## disk_types :: 디스크 유형(Disk Type, Persistent Disk)을 정의한다.
disk_types:
- disk_size: 1024
  name: default
- disk_size: 1024
  name: 1GB
  
... ((생략)) ...

## networks :: 네트워크(Network)를 정의한다. (AWS 경우, Subnet 및 Security Group, DNS, Gateway 등 설정)
networks:
- name: default
  subnets:
  - az: z1
    cloud_properties:
      security_groups: paasta-v50-security
      subnet: subnet-XXXXXXXXXXXXXXXXX
    dns:
    - 8.8.8.8
    gateway: 10.0.1.1
    range: 10.0.1.0/24
    reserved:
    - 10.0.1.2 - 10.0.1.9
    static:
    - 10.0.1.10 - 10.0.1.120

... ((생략)) ...

## vm_extentions :: 임의의 특정 IaaS 구성을 지정하는 가상머신 구성을 정의한다. (Security Groups 및 Load Balancers 등)
vm_extensions:
- name: cf-router-network-properties
- name: cf-tcp-router-network-properties
- name: diego-ssh-proxy-network-properties
- name: cf-haproxy-network-properties
- cloud_properties:
    ephemeral_disk:
      size: 51200
      type: gp2
  name: 50GB_ephemeral_disk

... ((생략)) ...

## vm_type :: 가상머신 유형(VM Type)을 정의한다. (AWS 경우, Instance type 설정)
vm_types:
- cloud_properties:
    ephemeral_disk:
      size: 3000
      type: gp2
    instance_type: t2.small
  name: minimal
- cloud_properties:
    ephemeral_disk:
      size: 10000
      type: gp2
    instance_type: t2.small
  name: small
  
... ((생략)) ...
```

- AZs

PaaS-TA에서 제공되는 Cloud Config 예제는 z1 ~ z6까지 설정되어 있다.  
z1 ~ z3까지는 PaaS-TA AP min VM이 설치되는 Zone이며, z4 ~ z6까지는 서비스가 설치되는 Zone으로 정의한다.   
3개 단위로 설정하는 이유는 서비스 3중화를 위해서이며, 설치하는 환경에 따라 다르게 설정해도 무방하다.  

- VM Types

VM Type은 IaaS에서 정의된 VM Type이다.  

※ 다음은 AWS에서 정의한 Instance Type이다.
![PaaSTa_FLAVOR_Image]

- Compilation

PaaS-TA AP min 및 서비스 설치 시, BOSH는 Compile 작업용 VM을 생성하여 소스를 컴파일하고, 이후 VM을 생성하여 컴파일된 파일을 대상 VM에 설치한 뒤 Compile 작업용 VM은 삭제된다. (Worker 수는 Compile VM의 수로, 많을수록 컴파일 속도가 빨라진다.)  

- Disk Size

PaaS-TA AP min 및 서비스가 설치되는 VM의 Persistent Disk Size이다.

- Networks

Networks는 AZ 별 Subnet Network, DNS, Security Groups, Network ID를 정의한다.  
보통 AZ 별로 256개의 IP를 정의할 수 있도록 Range Cider를 정의한다.

<br>

- Cloud Config 업데이트

```
$ bosh -e ${BOSH_ENVIRONMENT} update-cloud-config ~/workspace/paasta-deployment-min/cloud-config/{iaas}-cloud-config.yml
```

- Cloud Config 확인

```
$ bosh -e ${BOSH_ENVIRONMENT} cloud-config  
```

<br>

## <div id='2.6'/>2.6.  PaaS-TA AP min 설치 파일

common_vars.yml파일과 vars.yml을 수정하여 PaaS-TA AP min 설치시 적용하는 변수를 설정할 수 있다.

<table>
<tr>
<td>common_vars.yml</td>
<td>PaaS-TA AP 및 각종 Service 설치시 적용하는 공통 변수 설정 파일</td>
</tr>
<tr>
<td>min-vars.yml</td>
<td>PaaS-TA AP min 설치시 적용하는 변수 설정 파일</td>
</tr>
<tr>
<td>deploy-aws-4vms.sh</td>
<td>AWS 환경에 PaaS-TA AP min 4vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>deploy-aws-7vms.sh</td>
<td>AWS 환경에 PaaS-TA AP min 7vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>deploy-openstack-4vms.sh</td>
<td>OpenStack 환경에 PaaS-TA AP min 4vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>deploy-openstack-7vms.sh</td>
<td>OpenStack 환경에 PaaS-TA AP min 7vm 설치를 위한 Shell Script 파일</td>
</tr>
<td>deploy-vsphere-4vms.sh</td>
<td>vSphere 환경에 PaaS-TA AP min 4vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>deploy-vsphere-7vms.sh</td>
<td>vSphere 환경에 PaaS-TA AP min 7vm 설치를 위한 Shell Script 파일</td>
</tr>
<td>deploy-azure-4vms.sh</td>
<td>Azure 환경에 PaaS-TA AP min 4vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>deploy-azure-7vms.sh</td>
<td>Azure 환경에 PaaS-TA AP min 7vm 설치를 위한 Shell Script 파일</td>
</tr>
<td>deploy-gcp-4vms.sh</td>
<td>GCP 환경에 PaaS-TA AP min 4vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>deploy-gcp-7vms.sh</td>
<td>GCP 환경에 PaaS-TA AP min 7vm 설치를 위한 Shell Script 파일</td>
</tr>
<tr>
<td>min-paasta-deployment.yml</td>
<td>PaaS-TA AP min을 배포하는 Manifest 파일</td>
</tr>
</table>

<br>

### <div id='2.6.1'/>2.6.1. PaaS-TA AP min 설치 Variable File


- common_vars.yml  

~/workspace/common 폴더에 있는 [common_vars.yml](https://github.com/PaaS-TA/common/blob/master/common_vars.yml)에는 PaaS-TA AP min 및 각종 Service 설치 시 적용하는 공통 변수 설정 파일이 존재한다.  
PaaS-TA AP min을 설치 시 system_domain, paasta_admin_username, paasta_admin_password, paasta_database_port, paasta_cc_db_password, paasta_uaa_db_password, uaa_client_admin_secret, uaa_client_portal_secret의 값을 변경 하여 설치 할 수 있다.

> $ vi ~/workspace/common/common_vars.yml

```
... ((생략)) ...

system_domain: "xx.xx.xxx.xxx.nip.io"			# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
paasta_admin_username: "admin"				# PaaS-TA Admin Username
paasta_admin_password: "admin"				# PaaS-TA Admin Password
paasta_database_port: 5524				# PaaS-TA Database Port (e.g. 5524(postgresql)/13307(mysql)) -- Do Not Use "3306"&"13306" in mysql
paasta_cc_db_password: "cc_admin"			# CCDB Password(e.g. "cc_admin")
paasta_uaa_db_password: "uaa_admin"			# UAADB Password(e.g. "uaa_admin")
uaa_client_admin_secret: "admin-secret"			# UAAC Admin Client에 접근하기 위한 Secret 변수
uaa_client_portal_secret: "clientsecret"		# UAAC Portal Client에 접근하기 위한 Secret 변수

... ((생략)) ...
```

- min-vars.yml

PaaS-TA AP min 설치 할 때 적용되는 각종 변수값이나 배포 될 VM의 설정을 변경할 수 있다.

> $ vi ~/workspace/paasta-deployment-min/paasta/min-vars.yml
```
# SERVICE VARIABLE
deployment_name: "paasta"		# Deployment Name
network_name: "default"			# VM에 별도로 지정하지 않는 Default Network Name
releases_dir: "/home/ubuntu/workspace/paasta-5.5.1/release"	# Release Directory (offline으로 릴리즈 다운받아 사용시 설정)
haproxy_public_ip: "52.78.32.153"	# HAProxy IP (Public IP, HAproxy VM 배포시 필요)
haproxy_public_network_name: "vip"	# PaaS-TA Public Network Name
haproxy_private_network_name: "private" # PaaS-TA Private Network Name (vSphere use-haproxy-public-network-vsphere.yml 포함 배포시 설정 필요)
cc_db_encryption_key: "db-encryption-key"	# Database Encryption Key (Version Upgrade 시 동일 KEY 필수)
cert_days: 3650				# PaaS-TA 인증서 유효기간
private_ip: "10.244.0.34"   # Proxy IP (Private IP, BOSH-LITE 사용시 설정 필요)
uaa_login_logout_redirect_parameter_disable: "false"	
uaa_login_logout_redirect_parameter_whitelist: ["http://portal-web-user.15.165.2.88.xip.io","http://portal-web-user.15.165.2.88.xip.io/callback","http://portal-web-user.15.165.2.88.xip.io/login"]	# 포탈 페이지 이동을 위한 UAA Redirect Whitelist 등록 변수
uaa_login_branding_company_name: "PaaS-TA R&D"	# UAA 페이지 타이틀 명
uaa_login_branding_footer_legal_text: "Copyright © PaaS-TA R&D Foundation, Inc. 2017. All Rights Reserved."	# UAA 페이지 하단 영역 텍스트 
uaa_login_branding_product_logo: "iVBORw0KGgoAAAANSUhEUgAAAM0AAAAdCAYAAAAJguhGAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAyJpVFh0WE1MOmNvbS5hZG9iZS54bXAAAAAAADw/eHBhY2tldCBiZWdpbj0i77u/IiBpZD0iVzVNME1wQ2VoaUh6cmVTek5UY3prYzlkIj8+IDx4OnhtcG1ldGEgeG1sbnM6eD0iYWRvYmU6bnM6bWV0YS8iIHg6eG1wdGs9IkFkb2JlIFhNUCBDb3JlIDUuMy1jMDExIDY2LjE0NTY2MSwgMjAxMi8wMi8wNi0xNDo1NjoyNyAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNjcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RSZWY9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZVJlZiMiIHhtcDpDcmVhdG9yVG9vbD0iQWRvYmUgUGhvdG9zaG9wIENTNiAoV2luZG93cykiIHhtcE1NOkluc3RhbmNlSUQ9InhtcC5paWQ6QUNDMTA1MTZCRDNBMTFFNjkzMTVEQjMxRkE5QjkxNUMiIHhtcE1NOkRvY3VtZW50SUQ9InhtcC5kaWQ6QUNDMTA1MTdCRDNBMTFFNjkzMTVEQjMxRkE5QjkxNUMiPiA8eG1wTU06RGVyaXZlZEZyb20gc3RSZWY6aW5zdGFuY2VJRD0ieG1wLmlpZDpBQ0MxMDUxNEJEM0ExMUU2OTMxNURCMzFGQTlCOTE1QyIgc3RSZWY6ZG9jdW1lbnRJRD0ieG1wLmRpZDpBQ0MxMDUxNUJEM0ExMUU2OTMxNURCMzFGQTlCOTE1QyIvPiA8L3JkZjpEZXNjcmlwdGlvbj4gPC9yZGY6UkRGPiA8L3g6eG1wbWV0YT4gPD94cGFja2V0IGVuZD0iciI/Piy2YkgAAA9pSURBVHja7FwJeBRF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"	 # UAA 페이지 로고 이미지 (Base64)
uaa_login_branding_square_logo: "iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAyhpVFh0WE1MOmNvbS5hZG9iZS54bXAAAAAAADw/eHBhY2tldCBiZWdpbj0i77u/IiBpZD0iVzVNME1wQ2VoaUh6cmVTek5UY3prYzlkIj8+IDx4OnhtcG1ldGEgeG1sbnM6eD0iYWRvYmU6bnM6bWV0YS8iIHg6eG1wdGs9IkFkb2JlIFhNUCBDb3JlIDUuNi1jMTMyIDc5LjE1OTI4NCwgMjAxNi8wNC8xOS0xMzoxMzo0MCAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNjcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wTU09Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9tbS8iIHhtbG5zOnN0UmVmPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvc1R5cGUvUmVzb3VyY2VSZWYjIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtcE1NOkRvY3VtZW50SUQ9InhtcC5kaWQ6QkIwMjA5M0U5NEQ0MTFFNjk1M0FFQ0UxNkIxNEZFNjciIHhtcE1NOkluc3RhbmNlSUQ9InhtcC5paWQ6QkIwMjA5M0Q5NEQ0MTFFNjk1M0FFQ0UxNkIxNEZFNjciIHhtcDpDcmVhdG9yVG9vbD0iQWRvYmUgUGhvdG9zaG9wIENDIDIwMTUuNSAoV2luZG93cykiPiA8eG1wTU06RGVyaXZlZEZyb20gc3RSZWY6aW5zdGFuY2VJRD0ieG1wLmlpZDpEMzRGNDdCNTgxNEIxMUU2QjJFODk1MEQzM0EzNkMxOSIgc3RSZWY6ZG9jdW1lbnRJRD0ieG1wLmRpZDpEMzRGNDdCNjgxNEIxMUU2QjJFODk1MEQzM0EzNkMxOSIvPiA8L3JkZjpEZXNjcmlwdGlvbj4gPC9yZGY6UkRGPiA8L3g6eG1wbWV0YT4gPD94cGFja2V0IGVuZD0iciI/Psx4+gAAAASbSURBVHja7FZ9aFVlGP8973vOud9Tt7nl1lzNOWailKkV/iEohYn2Vx/QHxKCQkUUmEFhREXQvxJGhSFRlERFhEk1K9PUjJk0dZJzzuGc+3B3997tfp3zfvScu0X4T1AQ/nPP5eGce17e5/l9PO9zL1lrcTMvgZt8VQFUAVQBVAE4KHbDZIYWq1P7dpiJwQ7HdYxZsOyiXLzmYxGvOwJZD6ICDFxQKQeKNgEyi6A8BYcUSE0CJKCpDkExikgyBmTLQJzXhIaVSejxfsjaZtDkeaD3M9iVu6ByKbiNc+GYsUtt5R/fPIzxoSaSUZhIADM9sN5c/WG707Z2r+jc8gJFvAz0/zOyhTrf9RImBptsOQ4bSA4P0Alma8hcOLAtOPnaT7pwbQXJ2H+vQsRaRzkis8+ci5wZAHb4+EYpErBFbgdWzpYJ1p8NHQeNnVuuj7/epca6N89spH9ZOAb+vZsf9B18Juj97osgkznk9+zfY0ojbeEaFd9tT0OV55lhycwJFGOpoxbkcoSAIxZClmHiXkCdD7/qLtr6BkSa/Z74xx6gFBOwvtSXf9mp+r5+mrJDt5IRMNIB+T5MtHHUXfvyFvnKltUbbP7KbRRzoDMSxHBp9hNetnLnTaE/6d/XmdLwUjuvuRtOKiNCALpQYWopDqMcOB43q/Zggusd6rf3PjF9X26ncr5GOGyxGwUZJiq4TmkyqXOjHXLXi8/l7eCRR0WUi3gWekowKxGaMyvjXzd+z0wp27tUjZ18TFiboljdALleNlTAEjdw4MKJp2CvdW/2T+/5VEz03FmRee4S8NECpljSUqTiv5U+53N9spM/o3TsrYN06dCDSDCLPCcb4apBaAdXjhkg3OOZmYgISLfMCdiWRPN1arz7Qzm//aCNd5wIVEPeyXZvs6d2v0N+UZhIDUztWqgreYjRiywk53QjHFzD9oI6H/mIrD4Pk0svKH+zo8tJX1hqonwCmLyeZFlzrET4pVJYQ3isgxf2iOZ3mokXWBkNlWyHbX3oLBVL31P//mcpKMFG50PVb4Y+fQZycoDJJBkAd75gFeU4bFPT2ei6nRvJls9W5qEZOrYoOLHvADIXOxHzWB62RGnoosfyJhlAimW07CNHRCK0TM6tBSX4BAlen7gASp8GBQo62XLF1G4S9ujRZuQGYNkWwX1C0Qh0jeu7i5Z/7tz7xPNCmOEZAIbP3/QgjK9a/F/3vY+x7vtlCMrjQoI3Oi4XTUE5c2CcGsDlo8XKRHhC2uJViOkxBltkLdirhnu+dToff0qd6LnDz11/wHHVLcKUha5bOCXdfA9aFh7y6u46g4aFQG4QfwPI9LGsNbD5aaHT57bq/sNPYuryCjLTDEJUpLbcmCTNjC0cllvFWrYHcWbd2idvX/+2bNq0m8HY4PhRFNrXYI6bBQpp2LZVMP2HYVIJuBEe541tQP4qnBsGh1FhIeO0rN6L1pUfmNFz99HIHxtsYXCZ1qoO0taTKgs+a4xAjguZmaLaVSNINHS5NUu+Eqn6HFTAeYIQGdtRRGW6KR/w+VkrXtM3zqrq3/IqgCqAKoCbDeBPAQYAvdcfKsxKtoUAAAAASUVORK5CYII="	# UAA 페이지 타이틀 로고 이미지 (Base64)
uaa_login_links_passwd: "http://portal-web-user.15.165.2.88.xip.io/resetpasswd"	# UAA 페이지에서 Reset Password 누를 시 이동하는 링크 주소
uaa_login_links_signup: "http://portal-web-user.15.165.2.88.xip.io/createuser"	# UAA 페이지에서 Create Account 누를 시 이동하는 링크 주소
uaa_client_portal_redirect_uri: "http://portal-web-user.15.165.2.88.xip.io,http://portal-web-user.15.165.2.88.xip.io/callback"	# UAA Portal Client의 Redirect URI 지정 변수, 포탈에서 로그인 버튼 클릭 후 UAA 페이지에서 성공적으로 로그인했을 경우 이동하는 URI 경로

syslog_custom_rule: 'if ($msg contains "DEBUG") then stop'      # [MONITORING] PaaS-TA Logging Agent에서 전송할 Custom Rule
syslog_fallback_servers: []             # [MONITORING] PaaS-TA Syslog Fallback Servers


# STEMCELL
stemcell_os: "ubuntu-bionic"		# Stemcell OS
stemcell_version: "1.171"		# Stemcell Version

# SMOKE-TEST
smoke_tests_azs: ["z1"]			# Smoke-Test 가용 존
smoke_tests_instances: 1		# Smoke-Test 인스턴스 수
smoke_tests_vm_type: "minimal"		# Smoke-Test VM 종류
smoke_tests_network: "default"		# Smoke-Test 네트워크

# ROTATE-CC-DATABASE-KEY
rotate_cc_database_key_azs: ["z1"] 	# Rotate-CC-Database-Key 가용 존
rotate_cc_database_key_instances: 1 	# Rotate-CC-Database-Key 인스턴스 수
rotate_cc_database_key_vm_type: "minimal" # Rotate-CC-Database-Key VM 종류
rotate_cc_database_key_network: "default" # Rotate-CC-Database-Key 네트워크



## 4VM

# DATABASE
database_azs: ["z1"]      		# Database 가용 존
database_instances: 1     		# Database 인스턴스 수
database_vm_type: "small"   		# Database VM 종류
database_network: "default"   		# Database 네트워크
database_persistent_disk_type: "100GB"  # Database 영구 Disk 종류

# CONTROL
control_azs: ["z1"]  	    		# Control 가용 존
control_instances: 1     		# Control 인스턴스 수
control_vm_type: "small-highmem-16GB"   # Control VM 종류
control_network: "default"   		# Control 네트워크
control_vm_extensions: ["diego-ssh-proxy-network-properties"] # Control VM 확장

# ROUTER
router_azs: ["z1"]    			# Router 가용 존
router_instances: 1     		# Router 인스턴스 수
router_vm_type: "minimal"   		# Router VM 종류
router_network: "default"   		# Router 네트워크
router_vm_extensions: ["cf-router-network-properties"]  # Router VM 확장

# COMPUTE
compute_azs: ["z1"]    			# COMPUTE 가용 존
compute_instances: 1     		# COMPUTE 인스턴스 수
compute_vm_type: "small-highmem-16GB"  	# COMPUTE VM 종류
compute_network: "default"   		# COMPUTE 네트워크
compute_vm_extensions: ["100GB_ephemeral_disk"]  # COMPUTE VM 확장


## 7VM

# SINGLETON-BLOBSTORE
singleton_blobstore_azs: ["z1"]   	# Singleton-Blobstore 가용 존
singleton_blobstore_instances: 1  	# Singleton-Blobstore 인스턴스 수
singleton_blobstore_vm_type: "small"  	# Singleton-Blobstore VM 종류
singleton_blobstore_network: "default"  # Singleton-Blobstore 네트워크
singleton_blobstore_persistent_disk_type: "100GB" # Singleton-Blobstore 영구 Disk 종류

# TCP-ROUTER
tcp_router_azs: ["z1"]    		# TCP-Router 가용 존
tcp_router_instances: 1     		# TCP-Router 인스턴스 수
tcp_router_vm_type: "minimal"   	# TCP-Router VM 종류
tcp_router_network: "default"   	# TCP-Router 네트워크
tcp_router_vm_extensions: ["cf-tcp-router-network-properties"]  # TCP-Router VM 확장

# HAPROXY
haproxy_azs: ["z7"]     		# HAProxy 가용 존
haproxy_instances: 1      		# HAProxy 인스턴스 수
haproxy_vm_type: "minimal"    		# HAProxy VM 종류
haproxy_network: "default"    		# HAProxy 네트워크
```

이하는 UAA와 관련된 변수의 설명이다.

1. uaa_login_logout_redirect_parameter_whitelist : 포탈 페이지 이동을 위한 UAA Redirect Whitelist 등록 변수

```
ex) uaa_login_logout_redirect_parameter_whitelist=["{PaaS-TA PORTAL URI}","{PaaS-TA PORTAL URI}/callback","{PaaS-TA PORTAL URI}/login"]
```

2. uaa_login_links_signup : UAA 페이지에서 Create Account 버튼 클릭 시 이동하는 링크 주소

```
ex) uaa_login_links_signup="{PaaS-TA PORTAL URI}/createuser"
```

<img src="https://github.com/PaaS-TA/Guide-5.0-Ravioli/blob/master/install-guide/paasta/images/uaa-login-2.png">

3. uaa_login_links_passwd : UAA 페이지에서 Reset Password 버튼 클릭 시 이동하는 링크 주소

```
ex) uaa_login_links_passwd="{PaaS-TA PORTAL URI}/resetpasswd"
```

<img src="https://github.com/PaaS-TA/Guide-5.0-Ravioli/blob/master/install-guide/paasta/images/uaa-login.png" width="663px">


4. uaa_client_portal_redirect_uri : UAAC Portal Client의 Redirect URI 지정 변수, 포탈에서 로그인 버튼 클릭 후 UAA 페이지에서 로그인 성공 시 이동하는 URI

```
ex) uaa_client_portal_redirect_uri="{PaaS-TA PORTAL URI}, {PaaS-TA PORTAL URI}/callback"
```

5. uaa_client_portal_secret : UAAC Portal Client에 접근하기 위한 Secret 변수

```
ex) uaa_client_portal_secret="portalclient"
```

6. uaa_client_admin_secret : UAAC Admin Client에 접근하기 위한 Secret 변수

```
ex) uaa_client_admin_secret="admin-secret"
```

PaaS-TA AP min을 설치 후 UAAC의 활용 방법은 사용 가이드에 기타 CLI를 참고한다.

<br>

### <div id='2.6.2'/>2.6.2. PaaS-TA AP min Operation 파일

<table>
<tr>
<td>파일명</td>
<td>설명</td>
<td>요구사항</td>
</tr>
<tr>
<td>operations/min-use-postgres.yml</td>
<td>Database를 Postgres로 설치 <br>
    - min-use-postgres.yml 미적용 시 MySQL 설치  <br>
    - 3.5 이전 버전에서 Migration 시 필수  
</td>
<td></td>
</tr>
<tr>
<td>operations/min-use-haproxy.yml</td>
<td>HAProxy 적용 <br>
    - IaaS에서 제공하는 LB를 사용하여 PaaS-TA AP min 설치 시, Operation 파일을 제거하고 설치한다.
</td>
<td>Requires operation file: use-haproxy-public-network.yml <br>
    Requires value :  -v haproxy_private_ip
</td>
</tr>
<tr>
<td>operations/use-haproxy-public-network.yml</td>
<td>HAProxy Public Network 설정 <br>
    - IaaS에서 제공하는 LB를 사용하여 PaaS-TA AP min 설치 시, Operation 파일을 제거하고 설치한다.
</td>
<td>Requires: use-haproxy.yml <br>
    Requires Value :  <br>
    -v haproxy_public_ip <br>
    -v haproxy_public_network_name
</td>
</tr>
<tr>
<td>operations/min-use-router-public-network.yml</td>
<td>router를 외부 접근을 가능하게 수정한다.</td>
<td>4VMs 배포시 사용</td>
</tr>
<tr>
<td>operations/min-create-vm-singleton-blobstore.yml</td>
<td>4VMs에서 사용되는 database의 singleton-blobstore를 단일 VM으로 배포한다.</td>
<td>Requires: min-use-haproxy.yml <br>
7VMs 배포시 사용 <br>
Requires operation file: min-option-network-and-deployment.yml</td>
</tr>
<tr>
<td>operations/min-create-vm-tcp-router.yml</td>
<td>4VMs에서 사용되는 router의 tcp-router를 단일 VM으로 배포한다.</td>
<td>7VMs 배포시 사용</td>
</tr>
<tr>
<td>operations/min-cce.yml</td>
<td>CCE 조치를 적용하여 설치한다.</td>
<td></td>
</tr>
</table>

<br>

### <div id='2.6.3'/>2.6.3.   PaaS-TA AP min 설치 Shell Scripts
min-paasta-deployment.yml 파일은 PaaS-TA AP min를 배포하는 Manifest 파일이며, PaaS-TA AP min VM에 대한 설치 정의를 하게 된다.  

**※ PaaS-TA AP min 설치 시 명령어는 BOSH deploy를 사용한다. (IaaS 환경에 따라 Option이 다름)**

PaaS-TA AP min 배포 BOSH 명령어 예시

```
$ bosh -e ${BOSH_ENVIRONMENT} -d paasta deploy min-paasta-deployment.yml
```

PaaS-TA AP min 배포 시, 설치 Option을 추가해야 한다. 설치 Option에 대한 설명은 아래와 같다.

<table>
<tr>
<td>-e</td>
<td>BOSH Director 명</td>
</tr>
<tr>
<td>-d</td>
<td>Deployment 명 (기본값 paasta, 수정 시 다른 PaaS-TA 서비스에 영향을 준다.)</td>
</tr>   
<tr>
<td>-o</td>
<td>PaaS-TA 설치 시 적용하는 Option 파일로 IaaS별 속성, Haproxy 사용 여부, Database 설정 기능을 제공한다.
</td>
</tr>
<tr>
<td>-v</td>
<td>PaaS-TA 설치 시 적용하는 변수 또는 Option 파일에 변수를 설정할 경우 사용한다. <br> Option 파일 속성에 따라 필수 또는 선택 항목으로 나뉜다.</td>
</tr>
<tr>
<td>-l, --var-file</td>
<td>YAML파일에 작성한 변수를 읽어올때 사용한다.</td>
</tr>
</table>

- AWS 환경 4vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-aws-4vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-aws.yml \					# AWS 설정
	-o operations/min-use-router-public-network.yml \		# Router 외부 접근 설정
	-o operations/min-use-router-public-network-aws.yml \
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 적용
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- AWS 환경 7vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-aws-7vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-aws.yml \					# AWS 설정
        -o operations/min-create-vm-singleton-blobstore.yml \		# singleton-blobstore VM 배포
        -o operations/min-create-vm-tcp-router.yml \			# tcp-router VM 배포
        -o operations/min-use-haproxy.yml \				# HAProxy 적용
        -o operations/use-haproxy-public-network.yml \			# HAProxy Public Network 적용
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
        -o operations/min-option-network-and-deployment.yml \		# singleton-blobstore Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 설정
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- Openstack 환경 4vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-openstack-4vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-openstack.yml \					# Openstack 설정
	-o operations/min-use-router-public-network.yml \			# Router 외부 접근 설정
        -o operations/min-use-postgres.yml \					# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \			# Rename Network and Deployment
	-o operations/min-cce.yml \						# CCE 조치 적용
        -l min-vars.yml \							# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml						# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- Openstack 환경 7vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-openstack-7vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-openstack.yml \					# Openstack 설정
        -o operations/min-create-vm-singleton-blobstore.yml \		# singleton-blobstore VM 배포
        -o operations/min-create-vm-tcp-router.yml \			# tcp-router VM 배포
        -o operations/min-use-haproxy.yml \				# HAProxy 적용
        -o operations/use-haproxy-public-network.yml \			# HAProxy Public Network 적용
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
        -o operations/min-option-network-and-deployment.yml \		# singleton-blobstore Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 설정
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- vSphere 환경 4vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-vsphere-4vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
	-o operations/min-use-router-public-network.yml \			# Router 외부 접근 설정
        -o operations/min-use-router-public-network-vsphere.yml \
        -o operations/min-use-postgres.yml \					# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \			# Rename Network and Deployment
	-o operations/min-cce.yml \						# CCE 조치 적용
        -l min-vars.yml \							# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml						# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- vSphere 환경 7vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-vsphere-7vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-create-vm-singleton-blobstore.yml \		# singleton-blobstore VM 배포
        -o operations/min-create-vm-tcp-router.yml \			# tcp-router VM 배포
        -o operations/min-use-haproxy.yml \				# HAProxy 적용
        -o operations/use-haproxy-public-network-vsphere.yml \			# HAProxy Public Network 적용
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
        -o operations/min-option-network-and-deployment.yml \		# singleton-blobstore Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 설정
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- Azure 환경 4vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-azure-4vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-azure.yml \					        # Azure 설정
	-o operations/min-use-router-public-network.yml \			# Router 외부 접근 설정
        -o operations/min-use-postgres.yml \					# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \			# Rename Network and Deployment
	-o operations/min-cce.yml \						# CCE 조치 적용
        -l min-vars.yml \							# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml						# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- Azure 환경 7vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-azure-7vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-azure.yml \					# Azure 설정
        -o operations/min-create-vm-singleton-blobstore.yml \		# singleton-blobstore VM 배포
        -o operations/min-create-vm-tcp-router.yml \			# tcp-router VM 배포
        -o operations/min-use-haproxy.yml \				# HAProxy 적용
        -o operations/use-haproxy-public-network.yml \			# HAProxy Public Network 적용
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
        -o operations/min-option-network-and-deployment.yml \		# singleton-blobstore Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 설정
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- GCP 환경 4vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-gcp-4vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
	-o operations/min-use-router-public-network.yml \			# Router 외부 접근 설정
        -o operations/min-use-postgres.yml \					# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \			# Rename Network and Deployment
	-o operations/min-cce.yml \						# CCE 조치 적용
        -l min-vars.yml \							# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml						# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- GCP 환경 7vm 설치 시

```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-gcp-7vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-create-vm-singleton-blobstore.yml \		# singleton-blobstore VM 배포
        -o operations/min-create-vm-tcp-router.yml \			# tcp-router VM 배포
        -o operations/min-use-haproxy.yml \				# HAProxy 적용
        -o operations/use-haproxy-public-network.yml \			# HAProxy Public Network 적용
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
        -o operations/min-option-network-and-deployment.yml \		# singleton-blobstore Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 설정
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- Shell script 파일에 실행 권한 부여

```
$ chmod +x ~/workspace/paasta-deployment-min/paasta/*.sh
```

<br>

## <div id='2.7'/>2.7.  PaaS-TA AP min 설치
- 서버 환경에 맞추어 common_vars.yml와 min-vars.yml를 수정 한 뒤, Deploy 스크립트 파일의 설정을 수정한다.

- 4VM 배포 시
```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-aws-4vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-aws.yml \					# AWS 설정
	-o operations/min-use-router-public-network.yml \		# Router 외부 접근 설정
	-o operations/min-use-router-public-network-aws.yml \
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 적용
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```


- 7VM 배포 시
```
$ vi ~/workspace/paasta-deployment-min/paasta/deploy-aws-7vms.sh

BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -d paasta -n deploy min-paasta-deployment.yml \	# PaaS-TA-min Manifest File
        -o operations/min-aws.yml \					# AWS 설정
        -o operations/min-create-vm-singleton-blobstore.yml \		# singleton-blobstore VM 배포
        -o operations/min-create-vm-tcp-router.yml \			# tcp-router VM 배포
        -o operations/min-use-haproxy.yml \				# HAProxy 적용
        -o operations/use-haproxy-public-network.yml \			# HAProxy Public Network 적용
        -o operations/min-use-postgres.yml \				# Database Type 설정 (3.5버전 이하에서 Migration 시 필수)
        -o operations/min-rename-network-and-deployment.yml \		# Rename Network and Deployment
        -o operations/min-option-network-and-deployment.yml \		# singleton-blobstore Rename Network and Deployment
	-o operations/min-cce.yml \					# CCE 조치 설정
        -l min-vars.yml \						# PaaS-TA-min 설치시 적용하는 변수 설정 파일
        -l ../../common/common_vars.yml					# PaaS-TA 및 각종 Service 설치시 적용하는 공통 변수 설정 파일
```

- PaaS-TA AP min 설치 시 Shell Script 파일 실행 (BOSH 로그인 필요)

```
$ cd ~/workspace/paasta-deployment-min/paasta
$ ./deploy-{IaaS}-{VMs_Number}.sh
```

- PaaS-TA AP min 설치 확인

> $ bosh -e ${BOSH_ENVIRONMENT} vms -d paasta

```
ubuntu@inception:~$ bosh -e micro-bosh vms -d paasta
Using environment '10.0.1.6' as client 'admin'

Task 134. Done

Deployment 'paasta'

Instance                                       Process State  AZ  IPs          VM CID               VM Type             Active  Stemcell  
compute/e154dcdc-a2c1-4a85-86b7-607a02a30acf   running        z1  10.0.31.235  i-0f92f55575bf2567e  small-highmem-16GB  true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171 
control/a18f5e97-098c-47ab-9147-77f594571bd6   running        z1  10.0.31.234  i-053cd8f71d99f1a15  small-highmem-16GB  true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171 
database/7ea28d82-5d5b-471f-bde6-a65d4809062e  running        z1  10.0.31.233  i-0b2e54deaf0734f59  small               true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171 
router/c01b1aa4-43c9-42f6-9003-cf8f8664d142    running        z7  10.0.30.204  i-0a449def3351877b3  minimal             true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171 
                                                                  54.180.53.80                                                 

4 vms

Succeeded
```
```
ubuntu@inception:~$ bosh -e micro-bosh vms -d paasta
Using environment '10.0.1.6' as client 'admin'

Task 134. Done

Deployment 'paasta'

Instance                                                  Process State  AZ  IPs             VM CID               VM Type             Active  Stemcell  
compute/c3f53aed-469f-47ab-aa9b-94be30ca3687              running        z1  10.0.21.156     i-0617a496567bd859e  small-highmem-16GB  true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   
control/acd880a6-b309-452e-b996-0ef4252f8dd3              running        z1  10.0.21.153     i-0d9fbf3f662dec9a0  small-highmem-16GB  true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   
database/c92fd45f-1165-4d71-8df4-9b4270abdd0a             running        z1  10.0.21.151     i-0ead4f61c9be951b9  small               true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   
haproxy/5ccc73dd-cf7e-4f4c-a204-1e933eddfcf8              running        z7  10.0.20.151     i-02e5277d6fd829f34  minimal             true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   
                                                                             54.180.53.80                                             
router/4f58af5a-529c-41f7-866c-e2327978ea99               running        z1  10.0.21.154     i-0b5f2d42d2c2b9d06  minimal             true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   
singleton-blobstore/5ed376fe-1d84-45c8-a6e8-f938b7320a36  running        z1  10.0.21.152     i-08a432269ffb76663  small               true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   
tcp-router/f8fe5974-8340-4d16-ae02-0b7150828388           running        z1  10.0.21.155     i-04a845c8e7fc7cfb4  minimal             true    bosh-aws-xen-hvm-ubuntu-bionic-go_agent/1.171   


7 vms

Succeeded
```

<br>

## <div id='2.8'/>2.8.  PaaS-TA AP min 로그인

CF CLI를 설치하고 PaaS-TA AP min에 로그인한다.  
CF CLI는 v6과 v7중 선택해서 설치를 한다.  
CF API는 PaaS-TA AP min 배포 시 지정했던 System Domain 명을 사용한다.

- CF CLI v6 설치

```
$ wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
$ echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
$ sudo apt update
$ sudo apt install cf-cli -y
$ cf --version
```

- CF CLI v7 설치 (PaaS-TA AP min 5.1.0 이상)

```
$ wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
$ echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
$ sudo apt update
$ sudo apt install cf7-cli -y
$ cf --version
```

- CF API URL 설정

> $ cf api api.{system_domain} --skip-ssl-validation

```
ubuntu@inception:~$ cf api api.54.180.53.80.nip.io --skip-ssl-validation
Setting api endpoint to api.54.180.53.80.nip.io...
OK

api endpoint:   https://api.54.180.53.80.nip.io
api version:    3.87.0
```

- PaaS-TA AP min 로그인

> $ cf login

```
ubuntu@inception:~$ cf login
API endpoint: https://api.54.180.53.80.nip.io

Email> admin

Password>
Authenticating...
OK

Select an org (or press enter to skip):
```

[PaaSTa_FLAVOR_Image]:./images/ap/aws-vmtype.png


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > PaaS-TA AP - min
