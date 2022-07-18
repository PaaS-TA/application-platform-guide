### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Lifecycle Service

## Table of Contents

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)   
  1.3. [참고자료](#1.3)   

2. [라이프사이클 관리 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)     
  2.6. [서비스 설치 확인](#2.6)  
  
3. [라이프사이클 관리 서비스 관리 및 신청](#3)  
 3.1. [서비스 브로커 등록](#3.1)   
 3.2. [서비스 신청](#3.2)  
　3.2.1. [서비스 신청 - 포탈](#3.2.1)   
　3.2.2. [서비스 신청 - CLI](#3.2.2)   




## <div id="1"/> 1. 문서 개요

### <div id="1.1"/> 1.1. 목적

본 문서(라이프사이클 관리 서비스팩 설치 가이드)는 PaaS-TA에서 제공되는 서비스팩인 라이프사이클 관리 서비스팩을 Bosh를 이용하여 설치 하는 방법을 기술하였다.  

### <div id="1.2"/> 1.2. 범위

설치 범위는 라이프사이클 관리 서비스팩을 검증하기 위한 기본 설치를 기준으로 작성하였다.  

### <div id="1.3"/> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)   
라이프사이클 관리 서비스(TAIGA) 참고 자료  : [https://resources.taiga.io/](https://resources.taiga.io/)

## <div id="2"/> 2. 라이프사이클 관리 서비스 설치  

### <div id="2.1"/> 2.1. Prerequisite 

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.  
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.  

### <div id="2.2"/> 2.2. Stemcell 확인  

Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-bionic 1.90을 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version   OS             CPI  CID  
bosh-openstack-kvm-ubuntu-bionic-go_agent  1.90      ubuntu-bionic  -    ce507ae4-aca6-4a6d-b7c7-220e3f4aaa7d

(*) Currently deployed

1 stemcells

Succeeded
```

만약 해당 Stemcell이 업로드 되어 있지 않다면 [bosh.io 스템셀](https://bosh.io/stemcells/) 에서 해당되는 IaaS환경과 버전에 해당되는 스템셀 링크를 복사 후 다음과 같은 명령어를 실행한다.

```
# Stemcell 업로드 명령어 예제
$ bosh -e ${BOSH_ENVIRONMENT} upload-stemcell -n {STEMCELL_URL}
```

### <div id="2.3"/> 2.3. Deployment 다운로드

서비스 설치에 필요한 Deployment를 Git Repository에서 받아 서비스 설치 작업 경로로 위치시킨다.  

- Service Deployment Git Repository URL : https://github.com/PaaS-TA/service-deployment/tree/v5.1.11

```
# Deployment 다운로드 파일 위치 경로 생성 및 설치 경로 이동
$ mkdir -p ~/workspace
$ cd ~/workspace

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/service-deployment.git -b v5.1.11

# common_vars.yml 파일 다운로드(common_vars.yml가 존재하지 않는다면 다운로드)
$ git clone https://github.com/PaaS-TA/common.git

```

### <div id="2.4"/> 2.4. Deployment 파일 수정

BOSH Deployment manifest는 Components 요소 및 배포의 속성을 정의한 YAML 파일이다.  
Deployment 파일에서 사용하는 network, vm_type, disk_type 등은 Cloud config를 활용하고, 활용 방법은 PaaS-TA AP 설치 가이드를 참고한다.  

- Cloud config 설정 내용을 확인한다.   

> $ bosh -e micro-bosh cloud-config   

```
Using environment '10.0.1.6' as client 'admin'

azs:
- cloud_properties:
    availability_zone: ap-northeast-2a
  name: z1
- cloud_properties:
    availability_zone: ap-northeast-2a
  name: z2

... ((생략)) ...

disk_types:
- disk_size: 1024
  name: default
- disk_size: 1024
  name: 1GB

... ((생략)) ...

networks:
- name: default
  subnets:
  - az: z1
    cloud_properties:
      security_groups: paasta-security-group
      subnet: subnet-00000000000000000
    dns:
    - 8.8.8.8
    gateway: 10.0.1.1
    range: 10.0.1.0/24
    reserved:
    - 10.0.1.2 - 10.0.1.9
    static:
    - 10.0.1.10 - 10.0.1.120

... ((생략)) ...

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

Succeeded
```

- common_vars.yml을 서버 환경에 맞게 수정한다. 
- Lifecycle 서비스에서 사용하는 변수는 bosh_url, bosh_client_admin_id, bosh_client_admin_secret, bosh_director_port, bosh_oauth_port이다.

> $ vi ~/workspace/common/common_vars.yml
```
... ((생략)) ...

bosh_url: "https://10.0.1.6"			# BOSH URL (e.g. "https://00.000.0.0")
bosh_client_admin_id: "admin"			# BOSH Client Admin ID
bosh_client_admin_secret: "ert7na4jpew48"	# BOSH Client Admin Secret('echo $(bosh int ~/workspace/paasta-deployment/bosh/{iaas}/creds.yml --path /admin_password)' 명령어를 통해 확인 가능)
bosh_director_port: 25555			# BOSH director port
bosh_oauth_port: 8443				# BOSH oauth port

... ((생략)) ...

```


- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/service-deployment/lifecycle-service/vars.yml

```
# STEMCELL
stemcell_os: "ubuntu-bionic"                                                         # stemcell os
stemcell_version: "1.90"                                                           # stemcell version

# VM_TYPE
vm_type_default: "medium"                                                            # vm type default
vm_type_highmem: "small-highmem-16GB"                                                # vm type highmemory

# NETWORK
private_networks_name: "default"                                                     # private network name
public_networks_name: "vip"                                                          # public network name :: The public network name can only use "vip" or "service_public".

# MARIA_DB
mariadb_azs: [z3]                                                                    # mariadb : azs
mariadb_instances: 1                                                                 # mariadb : instances (1) 
mariadb_persistent_disk_type: "10GB"                                                 # mariadb : persistent disk type 
mariadb_port: "<MARIADB_PORT>"                                                       # mariadb : database port (e.g. 31306) -- Do Not Use "3306"
mariadb_admin_password: "<MARIADB_ADMIN_PASSWORD>"                                   # mariadb : database admin password (e.g. "paas-ta!admin")
mariadb_broker_username: "<MARIADB_BROKER_USERNAME>"                                 # mariadb : service-broker-user id (e.g. "applifecycle")
mariadb_broker_password: "<MARIADB_BROKER_PASSWORD>"                                 # mariadb : service-broker-user password (e.g. "broker!admin")

# SERVICE-BROKER
broker_azs: [z3]                                                                     # service-broker : azs
broker_instances: 1                                                                  # service-broker : instances (1)
broker_port: "<SERVICE_BROKER_PORT>"                                                 # service-broker : broker port (e.g. "8080")
broker_logging_level_broker: "INFO"                                                  # service-broker : broker logging level
broker_logging_level_hibernate: "INFO"                                               # service-broker : hibernate logging level
broker_services_id: "<SERVICE_BROKER_SERVICES_GUID>"                                 # service-broker : service guid (e.g. "b988f110-2bc3-46ce-8e55-9b8d50e529d4")
broker_services_plans_id: "<SERVICE_BROKER_SERVICES_PLANS_GUID>"                     # service-broker : service plan id (e.g. "6eb97b3e-91db-4880-ad8a-503003e8e7dd")

# APP-LIFECYCLE
app_lifecycle_azs: [z7]                                                              # app-lifecycle : azs
app_lifecycle_instances: 2                                                           # app-lifecycle : instances (N)
app_lifecycle_persistent_disk_type: "20GB"                                           # app-lifecycle : persistent disk type
app_lifecycle_public_ips: "<APP_LIFECYCLE_PUBLIC_IPS>"                               # app-lifecycle : public ips (e.g. ["00.00.00.00" , "11.11.11.11"])
app_lifecycle_admin_password: "<APP_LIFECYCLE_ADMIN_PASSWORD>"                       # app-lifecycle : app-lifecycle super admin password (e.g. "admin!super")
app_lifecycle_serviceadmin_password: "<APP_LIFECYCLE_SERVICEADMIN_INIT_PASSWORD>"    # app-lifecycle : app-lifecycle serviceadmin user init password (e.g. "Service!admin")
postgres_port: "<APP_LIFECYCLE_POSTGRES_PORT>"                                       # app-lifecycle : app-lifecycle postgres port (e.g. "5524" default:5432)
```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/service-deployment/lifecycle-service/deploy.sh

```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"    # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
CURRENT_IAAS="${CURRENT_IAAS}"                # IaaS Information (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 aws/azure/gcp/openstack/vsphere 입력)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"        # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d lifecycle-service deploy --no-redact lifecycle-service.yml \
    -o operations/${CURRENT_IAAS}-network.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml 
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/service-deployment/lifecycle-service
$ sh ./deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e micro-bosh -d lifecycle-service vms

```
Using environment '10.0.1.6' as client 'admin'

Task 108. Done

Deployment 'lifecycle-service'

Instance                                             Process State  AZ  IPs           VM CID               VM Type  Active  
app-lifecycle/4a4e1dc8-7214-46ca-9d3a-4254cb784e6f   running        z7  10.0.0.122    i-08fdee7878ea1fd77  medium   true  
                                                                        13.124.4.62                                   
app-lifecycle/a066d71a-0c93-48e3-bddc-0e6dadb1ffcd   running        z7  10.0.0.123    i-044fcd1932afeda19  medium   true  
                                                                        52.78.10.153                                  
mariadb/e859e63c-4358-4ef7-bbeb-93fa19be7baf         running        z3  10.0.81.122   i-0091a0c9848be5277  medium   true  
service-broker/3307c237-d9a9-4885-ae78-007db70a0e22  running        z3  10.0.81.123   i-00c9496182c78d040  medium   true  

4 vms

Succeeded
```

## <div id="3"/>3.  라이프사이클 관리 서비스 관리 및 신청

PaaS-TA 운영자 포탈을 통해 서비스를 등록하고 공개하면, PaaS-TA 사용자 포탈을 통해 서비스를 신청 하여 사용할 수 있다.

### <div id="3.1"/> 3.1. 서비스 브로커 등록

서비스의 설치가 완료 되면, PaaS-TA 포탈에서 서비스를 사용하기 위해 서비스 브로커를 등록해 주어야 한다.  
서비스 브로커 등록 시에는 개방형 클라우드 플랫폼에서 서비스 브로커를 등록 할 수 있는 권한을 가진 사용자로 로그인 되어 있어야 한다.

- 서비스 브로커 목록을 확인한다
> $ cf service-brokers

```
Getting service brokers as admin...

name   url
No service brokers found
```

- 서비스 브로커 등록 명령어
```
cf create-service-broker [SERVICE_BROKER] [USERNAME] [PASSWORD] [SERVICE_BROKER_URL]

[SERVICE_BROKER] : 서비스 브로커 명
[USERNAME] / [PASSWORD] : 서비스 브로커에 접근할 수 있는 사용자 ID / PASSWORD
[SERVICE_BROKER_URL] : 서비스 브로커 접근 URL
```

- 라이프사이클 관리 서비스 브로커를 등록한다.  

> $ cf create-service-broker app-lifecycle-service-broker admin cloudfoundry http://<service-broker_ip>:8080
```
### e.g. 라이프사이클 관리 서비스 브로커 등록
$ cf create-service-broker app-lifecycle-service-broker admin cloudfoundry http://10.0.81.123:8080
Creating service broker app-lifecycle-service-broker as admin...  
OK                                                               
```

- 등록된 라이프사이클 관리 서비스 브로커를 확인한다.
> $ cf service-brokers

```
Getting service brokers as admin...

name                           url
app-lifecycle-service-broker   http://10.0.81.123:8081
```

- 라이프사이클 관리 서비스의 서비스 접근 정보를 확인한다.
> $ cf service-access -b app-lifecycle-service-broker   

```
Getting service access for broker app-lifecycle-service-broker as admin...
broker: app-lifecycle-service-broker
   service         plan           access   orgs
   app-lifecycle   dedicated-vm   none
```

- 라이프사이클 관리 서비스의 서비스 접근 허용을 설정(전체)하고 서비스 접근 정보를 재확인 한다.
> $ cf enable-service-access app-lifecycle  
```
Enabling access to all plans of service app-lifecycle for all orgs as admin...
OK
```
> $ cf service-access -b app-lifecycle-service-broker  

```
Getting service access for broker app-lifecycle-service-broker as admin...
broker: app-lifecycle-service-broker
   service         plan           access   orgs
   app-lifecycle   dedicated-vm   all
```

### <div id='3.2'/> 3.2. 서비스 신청
#### <div id='3.2.1'/> 3.2.1. 서비스 신청 - 포탈

-	PaaS-TA 운영자 포탈에 접속하여 서비스를 등록한다.  

> ※ 운영관리 > 카탈로그 > 앱서비스 등록
> - 이름 : 라이프사이클 관리 서비스
> - 분류 :  개발 지원 도구
> - 서비스 : app-lifecycle
> - 썸네일 : [라이프사이클 관리 서비스 썸네일]
> - 문서 URL : https://github.com/PaaS-TA/PAAS-TA-APP-LIFECYCLE-SERVICE-BROKER
> - 서비스 생성 파라미터 : password / 패스워드
> - 앱 바인드 사용 : N
> - 공개 : Y
> - 대시보드 사용 : Y
> - 온디멘드 : N
> - 태그 : paasta / tag1, free / tag2
> - 요약 : 라이프사이클 관리 서비스
> - 설명 :
> 체계적인 Agile 개발 지원과 프로젝트 협업에 필요한 커뮤니케이션 중심의 문서 및 지식 공유 지원 기능을 제공하는 TAIGA를 dedicated 방식으로 제공합니다.
> 서비스 관리자 계정은 serviceadmin/<서비스 신청 시 입력한 Password> 입니다.
>  
> ![002]

-	PaaS-TA 사용자  포탈에 접속하여, 카탈로그를 통해 서비스를 신청한다.   

![003]

-	대시보드 URL을 통해 서비스에 접근한다.  (서비스의 관리자 계정은 serviceadmin/[서비스 신청시 입력받은 패스워드])  

![004]  

#### <div id='3.2.2'/> 3.2.2. 서비스 신청 - CLI
CLI 를 통한 Lifecycle 서비스 신청 방법을 설명한다.

- 서비스 인스턴스 신청 명령어
```
cf create-service [SERVICE] [PLAN] [SERVICE_INSTANCE]

[SERVICE] : Marketplace에서 보여지는 서비스 명
[PLAN] : 서비스에 대한 정책
[SERVICE_INSTANCE] : 생성할 서비스 인스턴스 이름
```

- Lifecycle 서비스를 신청한다. (password 변수 설정)
> $ cf create-service app-lifecycle dedicated-vm lifecycle -c '{"password":"password"}'
```
Creating service instance app-lifecycle in org system / space dev as admin...
OK
```

- 서비스 상세의 대시보드 URL 정보를 확인하여 서비스에 접근한다.
> $ cf service app-lifecycle
 ```
 ... (생략) ...
dashboard:        http://13.124.4.62/admin/|http://13.124.4.62/discover
service broker:   app-lifecycle-service-broker
 ... (생략) ...
 ```


[001]:./images/applifecycle-service/image001.png
[002]:./images/applifecycle-service/image002.png
[003]:./images/applifecycle-service/image003.png
[004]:./images/applifecycle-service/image004.png
[005]:./images/applifecycle-service/image005.png
[006]:./images/applifecycle-service/image006.png


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Lifecycle Service
