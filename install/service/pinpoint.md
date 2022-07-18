### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Pinpoint APM Service

## Table of Contents  

1. [문서 개요](#1)  
  1.1. [목적](#1.1)   
  1.2. [범위](#1.2)   
  1.3. [참고자료](#1.3)  
  
2. [Pinpoint 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)    
  2.6. [서비스 설치 확인](#2.6)  
 
3. [Sample Web App 연동 Pinpoint 연동](#3)    
  3.1. [Pinpoint 서비스 브로커 등록](#3.1)  
  3.2. [Sample Web App 구조](#3.2)   
  3.3. [PaaS-TA에서 서비스 신청](#3.3)  
  3.4. [Sample Web App에 서비스 바인드 신청 및 App 확인](#3.4)  

## <div id='1'> 1. 문서 개요
### <div id='1.1'> 1.1. 목적

본 문서(Pinpoint 서비스팩 설치 가이드)는 PaaS-TA에서 제공되는 서비스팩인 Pinpoint 서비스팩을 Bosh를 이용하여 설치 하는 방법을 기술하였다.

### <div id='1.2'> 1.2. 범위
설치 범위는 Pinpoint 서비스팩을 검증하기 위한 기본 설치를 기준으로 작성하였다.


### <div id='1.3'> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

## <div id='2'> 2. Pinpoint 서비스 설치

### <div id="2.1"/> 2.1. Prerequisite  

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.  
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.  

- bosh runtime-config를 확인하여 bosh-dns include deployments 에 pinpoint가 있는지 확인한다.  
 ※ bosh-dns include deployments에 pinpoint가 없다면 ~/workspace/paasta-deployment/bosh/runtime-configs 의 dns.yml 을 열어서 pinpoint를 추가하고, bosh runtime-config를 업데이트 해준다.    

> $ bosh -e micro-bosh runtime-config
```
Using environment '10.0.1.6' as client 'admin'

---
addons:
- include:
    deployments:
    - paasta
    - pinpoint
    - pinpoint-monitoring
    - rabbitmq
    stemcell:
    - os: ubuntu-trusty
    - os: ubuntu-xenial
    - os: ubuntu-bionic
  jobs:
  - name: bosh-dns
    properties:
      api:
        client:
          tls: "((/dns_api_client_tls))"
        server:
          tls: "((/dns_api_server_tls))"
      cache:
        enabled: true
      health:
        client:
          tls: "((/dns_healthcheck_client_tls))"
        enabled: true
        server:
          tls: "((/dns_healthcheck_server_tls))"
    release: bosh-dns
  name: bosh-dns
...(생략)...

Succeeded
```

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

- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/service-deployment/pinpoint/vars.yml
```
# STEMCELL
stemcell_os: "ubuntu-bionic"                                     # stemcell os
stemcell_version: "1.90"                                       # stemcell version

# NETWORK
private_networks_name: "default"                                 # private network name
public_networks_name: "vip"                                      # public network name

# H_MASTER
master_azs: [z3]                                                 # h_master azs
master_instances: 1                                              # h_master instances (default : 1)
master_vm_type: "large"                                          # h_master vm type 
master_persistent_disk_type: "30GB"                              # h_master persistent disk type

# COLLECTOR
collector_azs: [z3]                                              # collector azs
collector_instances: 1                                           # collector instances (default : 1)
collector_vm_type: "large"                                       # collector vm type
collector_persistent_disk_type: "30GB"                           # collector persistent disk type

collector_tcp_port: 29994                                        # collector tcp listen port (default : 29994)
collector_stat_port: 29995                                       # collector stat port (default : 29995)
collector_span_port: 29996                                       # collector span port (default : 29996)

# PINPOINT_WEB
pinpoint_azs: [z3]                                               # pinpoint azs
pinpoint_instances: 1                                            # pinpoint instances (default : 1)
pinpoint_vm_type: "large"                                        # pinpoint vm type
pinpoint_persistent_disk_type: "30GB"                            # pinpoint persistent disk type

# BROKER
broker_azs: [z3]                                                 # broker azs
broker_instances: 1                                              # broker instances (default : 1)
broker_vm_type: "large"                                          # broker vm type
broker_persistent_disk_type: "30GB"                              # broker persistent disk type

# HAPROXY_WEBUI
webui_azs: [z7]                                                  # webui azs
webui_instances: 1                                               # webui instances (default : 1)
webui_vm_type: "large"                                           # webui vm type
webui_persistent_disk_type: "30GB"                               # webui persistent disk type
webui_haproxy_public_ip: "<WEB_UI_PUBLIC_IP>"                    # webui haproxy's public IP
```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/service-deployment/pinpoint/deploy.sh

```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"	# common_vars.yml File Path (e.g. ../../common/common_vars.yml)
CURRENT_IAAS="${CURRENT_IAAS}"			# IaaS Information (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 aws/azure/gcp/openstack/vsphere 입력)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"		# bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d pinpoint deploy --no-redact pinpoint.yml \
    -o operations/${CURRENT_IAAS}-network.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/service-deployment/pinpoint  
$ sh ./deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e micro-bosh -d pinpoint vms  

```
Using environment '10.0.1.6' as client 'admin'

Task 5006. Done

Deployment 'pinpoint'

Instance                                              Process State  AZ  IPs            VM CID                                   VM Type  Active
broker/7a9d8423-c5f9-4e3c-be16-42e9215f8268           running        z5  10.30.107.182  vm-1b9b0891-8a02-45af-8b0c-ca21bf56e64e  minimal  true
collector/5f10f9cf-67ed-4c08-8c14-99d7f7a818c2        running        z5  10.30.107.145  vm-1714d225-85fa-4236-94db-7ed26baa52b3  minimal  true
h_master/723a34d6-0756-41e2-b11e-1530596cbb09         running        z5  10.30.107.175  vm-037e021b-3dba-4ea3-961c-6bd259d25c4b  minimal  true
haproxy_webui/22999f9c-0798-43a5-b93b-bc5d44c4d210    running        z5  10.30.107.178  vm-294e229c-aa89-4a5c-9960-4009579bbf54  minimal  true
									 3.12.24.53
webui/30f7c9cf-ab03-4f78-a9bf-96c5148a9ec1            running        z5  10.30.107.180  vm-62d5e876-2ab4-4327-ba54-2f74b53e3da9  minimal  true

5 vms

Succeeded
```

##  <div id='3'> 3. Sample Web App 연동 Pinpoint 연동

본 Sample Web App은 개방형 클라우드 플랫폼에 배포되며 Pinpoint의 서비스를 Provision과 Bind를 한 상태에서 사용이 가능하다.

### <div id='3.1'> 3.1. Pinpoint 서비스 브로커 등록

Pinpoint 서비스팩 배포가 완료 되었으면 Application에서 서비스 팩을 사용하기 위해서 먼저 Pinpoint 서비스 브로커를 등록해 주어야 한다.  
서비스 브로커 등록시 PaaS-TA에서 서비스브로커를 등록 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- 서비스 브로커 목록을 확인한다.

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

- Pinpoint 서비스 브로커를 등록한다.

> $ cf create-service-broker pinpoint-service-broker admin cloudfoundry http://<broker_ip>:8080

```
$ cf create-service-broker pinpoint-service-broker admin cloudfoundry http://10.30.107.182:8080
Creating service broker pinpoint-service-broker as admin...
OK
```

-   등록된 Pinpoint 서비스 브로커를 확인한다.

> $ cf service-brokers
```
Getting service brokers as admin...
name url
pinpoint-service-broker http://10.30.107.182:8080
```

-   접근 가능한 서비스 목록을 확인한다.

> $ cf service-access
```
Getting service access as admin...
broker: pinpoint-service-broker
   offering   plan                access   orgs
   Pinpoint   Pinpoint_standard   none  
```
서비스 브로커 생성시 디폴트로 접근을 허용하지 않는다.

- 특정 조직에 해당 서비스 접근 허용을 할당하고 접근 서비스 목록을 다시 확인한다. (전체 조직)

> $ cf enable-service-access Pinpoint
```
Enabling access to all plans of service Pinpoint for all orgs as admin...
OK
```

- 서비스 접근 허용을 확인한다.
> $ cf service-access
```
broker: pinpoint-service-broker
   offering   plan                access   orgs
   Pinpoint   Pinpoint_standard   all  
```

### <div id='3.2'> 3.2. Sample Web App 다운로드

Sample Web App은 PaaS-TA에 App으로 배포가 된다. 배포된 App에 Pinpoint 서비스 Bind 를 통하여 초기 데이터를 생성하게 된다.  
바인드 완료 후 연결 url을 통하여 브라우저로 해당 App에 대한 Pinpoint 서비스 모니터링을 할 수 있다.

- Sample App 묶음 다운로드
```
$ wget https://nextcloud.paas-ta.org/index.php/s/NDgriPk5cgeLMfG/download --content-disposition  
$ unzip paasta-service-samples.zip  
$ cd paasta-service-samples/pinpoint  
```

<br>

### <div id='3.3'> 3.3. PaaS-TA에서 서비스 신청

Sample Web App에서 Pinpoint 서비스를 사용하기 위해서는 서비스 신청(Provision)을 해야 한다.  
*참고: 서비스 신청시 PaaS-TA에서 서비스를 신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.  

- 먼저 PaaS-TA Marketplace에서 서비스가 있는지 확인을 한다.

> $ cf marketplace

```
Getting services from marketplace in org org / space space as admin...
OK

service    plans               description
Pinpoint   Pinpoint_standard   A simple pinpoint implementation
```

- 서비스 인스턴스 신청 명령어
```
cf create-service [SERVICE] [PLAN] [SERVICE_INSTANCE]

[SERVICE] : Marketplace에서 보여지는 서비스 명
[PLAN] : 서비스에 대한 정책
[SERVICE_INSTANCE] : 생성할 서비스 인스턴스 이름
```

- Marketplace에서 원하는 서비스가 있으면 서비스 신청(Provision)을 하여 서비스 인스턴스를 생성한다.

> $ cf create-service Pinpoint Pinpoint_standard PS1

```
Creating service instance PS1 in org org / space space as admin...
OK
```

- 생성된 Pinpoint 서비스 인스턴스를 확인한다.

> $ cf services
```
Getting services in org system / space space as admin...
OK

name   service      plan                 bound apps   last
PS1    Pinpoint     Pinpoint_standard                 create succeeded
```

### <div id='3.4'> 3.4. Sample Web App에 서비스 바인드 신청 및 App 확인

서비스 신청이 완료되었으면 Sample Web App 에서는 생성된 서비스 인스턴스를 Bind 하여 App에서 Pinpoint 서비스를 이용한다.  
*참고: 서비스 Bind 신청시 PaaS-TA 플랫폼에서 서비스 Bind신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- manifest 파일을 확인한다. 
	
> $ vi manifest.yml   

```
---
applications:
- name: spring-music-pinpoint
  memory: 1G
  disk_quota: 1G 
  random-route: false
  path: spring-music-1.0.jar
  buildpacks:
    - pinpoint_buildpack
  env:
    JBP_CONFIG_SPRING_AUTO_RECONFIGURATION: '{enabled: false}'
```

- Pinpoint buildpack 등록  
	
> $ cf create-buildpack pinpoint_buildpack java-buildpack-pinpoint-monitoring-2402a2c.zip 14
```	
Creating buildpack pinpoint_buildpack as admin...
OK

Uploading buildpack pinpoint_buildpack as admin...
 13.96 MiB / 13.96 MiB [=================================================================================================
OK

Processing uploaded buildpack pinpoint_buildpack...
OK
```

- --no-start 옵션으로 App을 배포한다.

> $ cf push --no-start 
```  
Applying manifest file /home/ubuntu/workspace/samples/paasta-service-samples/pinpoint/manifest.yml...
Manifest applied
Packaging files to upload...
Uploading files...
 48.91 MiB / 48.91 MiB [=================================================================================================

Waiting for API to complete processing files...

name:              spring-music-pinpoint
requested state:   stopped
routes:            spring-music-pinpoint.paastacloud.shop
last uploaded:     
stack:             
buildpacks:        

type:           web
sidecars:       
instances:      0/1
memory usage:   1024M
     state   since                  cpu    memory   disk     details
#0   down    2021-11-22T05:26:04Z   0.0%   0 of 0   0 of 0   
```  
	
- Sample Web App에서 생성한 서비스 인스턴스 바인드 신청을 한다.

> $ cf bind-service spring-music-pinpoint PS1 -c '{"application_name":"spring-music-pinpoint"}'
	
```	
Binding service PS1 to app spring-music-pinpoint in org org / space space as admin...
OK
TIP: Use 'cf restage spring-music-pinpoint' to ensure your env variable changes take effect
```
	
App 구동 시 Service와의 통신을 위하여 보안 그룹을 추가한다.
	
- rule.json을 편집한다.  
> $ vi rule.json   
```
## pinpoint의 collector IP를 destination에 설정
[
  {
    "protocol": "all",
    "destination": "<collector_ip>"
  }
]
```
  
- 보안 그룹을 생성한다.  

> $ cf create-security-group pinpoint rule.json  

```
Creating security group pinpoint as admin...

OK		
```
  
- Pinpoint 서비스를 사용할수 있도록 생성한 보안 그룹을 적용한다.
> $ cf bind-running-security-group pinpoint  
```
Binding security group pinpoint to running as admin...
OK		
```

- 바인드가 적용되기 위해서 App을 restage한다.

> $ cf restage spring-music-pinpoint

```	
Staging app and tracing logs...
   Downloading pinpoint_buildpack...
   Downloaded pinpoint_buildpack (14M)
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd creating container for instance 89996fa5-3197-4a9c-8304-9a5b288c74ee
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd successfully created container for instance 89996fa5-3197-4a9c-8304-9a5b288c
   Downloading app package...
   Downloaded app package (50M)

........
........
	
Instances starting...
Instances starting...

name:              spring-music-pinpoint
requested state:   started
routes:            spring-music-pinpoint.paasta.kr
last uploaded:     Mon 22 Nov 05:28:14 UTC 2021
stack:             cflinuxfs3
buildpacks:        
	name                 version   detect output   buildpack name
	pinpoint_buildpack                             

type:           web
sidecars:       
instances:      1/1
memory usage:   1024M
     state     since                  cpu    memory        disk           details
#0   running   2021-11-22T05:28:37Z   0.0%   47.5M of 1G   177.9M of 1G   

type:           task
sidecars:       
instances:      0/0
memory usage:   1024M
There are no running instances of this process.
```

- Service 정상 구동 확인
> $ cf service PS1
```
name:             PS1
service:          Pinpoint
tags:             
plan:             Pinpoint_standard
description:      A simple pinpoint implementation
documentation:    http://www.openpaas.org
dashboard:        http://3.53.24.53/#/main
service broker:   pinpoint-service-broker
```

- PINPOINT UI 접근 
```
# PINPOINT APP UI
http://[DASHBOARD]/[application_name]@SPRING_BOOT/realtime

e.g. http://3.12.24.53/#/main/spring-music-pinpoint@SPRING_BOOT/realtime
```	
	
![pinpoint_image_04]

[pinpoint_image_01]:./images/pinpoint/pinpoint-image1.png
[pinpoint_image_01-1]:./images/pinpoint/pinpoint-image1-1.png
[pinpoint_image_02]:./images/pinpoint/pinpoint-image2.png
[pinpoint_image_03]:./images/pinpoint/pinpoint-image3.png
[pinpoint_image_04]:./images/pinpoint/pinpoint-image4.png


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Pinpoint APM Service
