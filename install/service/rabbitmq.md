### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > RabbitMQ Service

## Table of Contents

1. [문서 개요](#1)   
  1.1. [목적](#1.1)   
  1.2. [범위](#1.2)   
  1.3. [참고자료](#1.3)   

2. [RabbitMQ 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)    
  2.6. [서비스 설치 확인](#2.6)  

3. [RabbitMQ 연동 Sample App 설명](#3)  
  3.1. [서비스 브로커 등록](#3.1)  
  3.2. [Sample App 다운로드](#3.2)  
  3.3. [PaaS-TA에서 서비스 신청](#3.3)  
  3.4. [Sample App에 서비스 바인드 신청 및 App 확인](#3.4)   
     
## <div id='1'> 1. 문서 개요
### <div id='1.1'> 1.1. 목적
본 문서(RabbitMQ 서비스팩 설치 가이드)는 PaaS-TA에서 제공되는 서비스팩인 RabbitMQ 서비스팩을 Bosh를 이용하여 설치 하는 방법을 기술하였다.  

### <div id='1.2'> 1.2. 범위 
설치 범위는 RabbitMQ 서비스팩을 검증하기 위한 기본 설치를 기준으로 작성하였다. 


### <div id='1.3'> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  


## <div id='2'> 2. RabbitMQ 서비스 설치

### <div id="2.1"/> 2.1. Prerequisite  

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.  
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.  

- bosh runtime-config를 확인하여 bosh-dns include deployments 에 rabbitmq가 있는지 확인한다.  
 ※ bosh-dns include deployments에 rabbitmq가 없다면 ~/workspace/paasta-deployment/bosh/runtime-configs 의 dns.yml 을 열어서 rabbitmq를 추가하고, bosh runtime-config를 업데이트 해준다.    

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
본 가이드의 Stemcell은 ubuntu-bionic 1.97을 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version   OS             CPI  CID  
bosh-openstack-kvm-ubuntu-bionic-go_agent  1.97      ubuntu-bionic  -    ce507ae4-aca6-4a6d-b7c7-220e3f4aaa7d

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

- Service Deployment Git Repository URL : https://github.com/PaaS-TA/service-deployment/tree/v5.1.14

```
# Deployment 다운로드 파일 위치 경로 생성 및 설치 경로 이동
$ mkdir -p ~/workspace
$ cd ~/workspace

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/service-deployment.git -b v5.1.14

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
- RabbitMQ에서 사용하는 변수는 system_domain, paasta_admin_username, paasta_admin_password, paasta_nats_ip 이다.

> $ vi ~/workspace/common/common_vars.yml
```
... ((생략)) ...

system_domain: "61.252.53.246.nip.io"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
paasta_admin_username: "admin"			# PaaS-TA Admin Username
paasta_admin_password: "admin"			# PaaS-TA Admin Password
paasta_nats_ip: "10.0.1.121"
  
... ((생략)) ...

```


- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/service-deployment/rabbitmq/vars.yml

```
deployment_name: "rabbitmq"                                  # rabbitmq deployment name 

# STEMCELL
stemcell_os: "ubuntu-bionic"                                # stemcell os
stemcell_version: "1.97"                                    # stemcell version

# VM_TYPE
vm_type_small: "minimal"                                    # vm type small 

# NETWORK
private_networks_name: "default"                            # private network name

# COMMON
bosh_name: "micro-bosh"                                     # bosh name (e.g. micro-bosh) -- ('bosh env' 명령어를 통해 확인 가능)
paasta_deployment_name: "paasta"                            # paasta application platform name (e.g. paasta)

# RABBITMQ
rabbitmq_azs: [z3]                                          # rabbitmq : azs
rabbitmq_instances: 1                                       # rabbitmq : instances (1) 
rabbitmq_private_ips: "<RABBITMQ_PRIVATE_IPS>"              # rabbitmq : private ips (e.g. "10.0.81.31")
management_username: "<MANAGEMENT_USERNAME>"  		    # rabbitmq : username (e.g. "madmin") *broker/administrator_username != management_username

# HAPROXY
haproxy_azs: [z3]                                           # haproxy : azs
haproxy_instances: 1                                        # haproxy : instances (1) 
haproxy_private_ips: "<HAPROXY_PRIVATE_IPS>"                # haproxy : private ips (e.g. "10.0.81.32")

# SERVICE-BROKER
broker_azs: [z3]                                            # service-broker : azs
broker_instances: 1                                         # service-broker : instances (1)
broker_port: 4567                                           # service-broker : broker port (e.g. "4567")
broker_username: "<SERVICE_BROKER_USERNAME>"		    # service-broker : username (e.g. "admin") *broker/administrator_username != management_username
broker_password: "<SERVICE_BROKER_PASSWORD>"                # service-broker : password (e.g. "admin" no recommand)
administrator_username: "<SERVICE_BROKER_ADMIN_USERNAME>"   # servier-broker : administrator username (e.g. "administrator")

# BROKER-REGISTRAR
broker_registrar_azs: [z3]                                  # broker-registrar : azs
broker_registrar_instances: 1                               # broker-registrar : instances (1) 

# BROKER-DEREGISTRAR
broker_deregistrar_azs: [z3]                                # broker-deregistrar : azs
broker_deregistrar_instances: 1                             # broker-deregistrar : instances (1)

```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/service-deployment/rabbitmq/deploy.sh

```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"  # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"      # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d rabbitmq deploy --no-redact rabbitmq.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/service-deployment/rabbitmq  
$ sh ./deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e micro-bosh -d rabbitmq vms  

```
Using environment '10.30.40.111' as user 'admin' (openid, bosh.admin)

Task 8077. Done

Deployment 'rabbitmq'

Instance                                                Process State  AZ  IPs            VM CID                                   VM Type  Active  
haproxy/a30fb543-000d-4f74-b62d-7418da0e6101            running        z5  10.30.107.192  vm-fbd4a04a-5346-4e00-b793-17c327f90aa7  minimal  true  
rmq-broker/52629ddb-32c9-4097-b9f6-e5dc0aff55ce         running        z5  10.30.107.191  vm-5238f05b-ec4f-449c-ab1d-a1a5b932d76e  minimal  true  
rmq/a4ef4c7e-4776-411d-8317-b2b059e416dd                running        z5  10.30.107.193  vm-f8d8a62d-bfc4-442e-8306-9f133ebfc518  minimal  true  

3 vms

Succeeded
```

## <div id='3'> 3. RabbitMQ 연동 Sample App 설명

본 Sample App은 PaaS-TA에 배포되며 RabbitMQ의 서비스를 Provision과 Bind를 한 상태에서 사용이 가능하다.

### <div id='3.1'> 3.1. 서비스 브로커 등록 

RabbitMQ 서비스팩 배포가 완료 되었으면 Application에서 서비스 팩을 사용하기 위해서 먼저 RabbitMQ 서비스 브로커를 등록해 주어야 한다.
서비스 브로커 등록시에는 PaaS-TA에서 서비스 브로커를 등록할 수 있는 사용자로 로그인 하여야 한다

- 서비스 브로커 목록을 확인한다.
> $ cf service-brokers
```
Getting service brokers as admin...
  
name   url
No service brokers found
```

<br>

- 서비스 브로커 등록 명령어
```
cf create-service-broker [SERVICE_BROKER] [USERNAME] [PASSWORD] [SERVICE_BROKER_URL]

[SERVICE_BROKER] : 서비스 브로커 명
[USERNAME] / [PASSWORD] : 서비스 브로커에 접근할 수 있는 사용자 ID / PASSWORD
[SERVICE_BROKER_URL] : 서비스 브로커 접근 URL
```
	
- rabbitmq 서비스 브로커를 등록한다.

> $ cf create-service-broker rabbitmq-service-broker admin cloudfoundry http://<rmq-broker_ip>:4567
```
$ cf create-service-broker rabbitmq-service-broker admin cloudfoundry http://10.30.107.191:4567
Creating service broker rabbitmq-service-broker as admin...
OK

```
<br>

- 등록된 RabbitMQ 서비스 브로커를 확인한다.

> $ cf service-brokers  

```
Getting service brokers as admin...

name                    url
rabbitmq-service-broker http://10.30.107.191:4567
```
<br>

- 접근 가능한 서비스 목록을 확인한다.

> $ cf service-access

```
Getting service access as admin...
broker: rabbitmq-service-broker
   service      plan       access   orgs
   rabbitmq     standard   none      
```

- 서비스 브로커 등록시 최초에는 접근을 허용하지 않는다. 따라서 access는 none으로 설정된다.

- 특정 조직에 해당 서비스 접근 허용을 할당하고 접근 서비스 목록을 다시 확인한다. (전체 조직)

> $ cf enable-service-access rabbitmq 

```
Enabling access to all plans of service rabbitmq for all orgs as admin...
OK
```

> $ cf service-access
```
Getting service access as admin...
broker: rabbitmq-service-broker
   service      plan       access   orgs
   rabbitmq     standard   all      
```

### <div id='3.2'> 3.2. Sample App 다운로드

- Sample App 묶음 다운로드
```
$ wget https://nextcloud.paas-ta.org/index.php/s/BoSbKrcXMmTztSa/download --content-disposition  
$ unzip paasta-service-samples-459dad9.zip  
$ cd paasta-service-samples/rabbitmq  
```

<br>


### <div id='3.3'> 3.3. 서비스 신청
Sample App에서 RabbitMQ 서비스를 사용하기 위해서는 서비스 신청(Provision)을 해야 한다.
*참고: 서비스 신청시 PaaS-TA에서 서비스를 신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- 먼저 PaaS-TA Marketplace에서 서비스가 있는지 확인을 한다.

> $ cf marketplace

```
getting services from marketplace in org system / space dev as admin...
OK

service      plans         description                                                                           broker
rabbitmq     standard      RabbitMQ is a robust and scalable high-performance multi-protocol messaging broker.   rabbitmq-service-broker

TIP: Use 'cf marketplace -s SERVICE' to view descriptions of individual plans of a given service.
```
<br>

- 서비스 인스턴스 신청 명령어
```
cf create-service [SERVICE] [PLAN] [SERVICE_INSTANCE]

[SERVICE] : Marketplace에서 보여지는 서비스 명
[PLAN] : 서비스에 대한 정책
[SERVICE_INSTANCE] : 생성할 서비스 인스턴스 이름
```

- Marketplace에서 원하는 서비스가 있으면 서비스 신청(Provision)을 한다.

> $ cf create-service rabbitmq standard my_rabbitmq_service
```
Creating service instance my_rabbitmq_service in org system / space dev as admin...
OK
```

<br>

- 생성된 rabbitmq 서비스 인스턴스를 확인한다.

> $ cf services

```
Getting services in org system / space dev as admin...

name                  service      plan       bound apps   last operation     broker                    upgrade available
my_rabbitmq_service   rabbitmq     standard                create succeeded   rabbitmq-service-broker   
```

<br>

### <div id='3.4'> 3.4. Sample App에 서비스 바인드 신청 및 App 확인
서비스 신청이 완료되었으면 cf 에서 제공하는 rabbit-example-app을 다운로드해서 테스트를 진행한다.
* 참고: 서비스 Bind 신청시 PaaS-TA에서 서비스 Bind 신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- manifest 파일을 확인한다.  

> $ vi manifest.yml   

```
---
applications:
- name: rabbit-example-app
  command: thin -R config.ru start
  path: .
  buildpacks:
  - ruby_buildpack
```

- --no-start 옵션으로 App을 배포한다.

> $ cf push --no-start 
```  
Applying manifest file /home/ubuntu/workspace/samples/paasta-service-samples/rabbitmq/manifest.yml...
Manifest applied
Packaging files to upload...
Uploading files...
 3.16 MiB / 3.16 MiB [===================================================================================================

Waiting for API to complete processing files...

name:              rabbit-example-app
requested state:   stopped
routes:            rabbit-example-app.paasta.kr
last uploaded:     
stack:             
buildpacks:        

type:            web
sidecars:        
instances:       0/1
memory usage:    1024M
start command:   thin -R config.ru start
     state   since                  cpu    memory   disk     details
#0   down    2021-11-22T05:32:24Z   0.0%   0 of 0   0 of 0   

```  
  
- Sample Web App에서 생성한 서비스 인스턴스 바인드 신청을 한다.

> $ cf bind-service rabbit-example-app my_rabbitmq_service 

```	
Binding service my_rabbitmq_service to app rabbit-example-app in org system / space dev as admin...
OK
```

App 구동 시 Service와의 통신을 위하여 보안 그룹을 추가한다.

- rule.json을 편집한다.  
> $ vi rule.json   
```
## rabbitmq의 haproxy IP를 destination에 설정
[
  {
    "protocol": "all",
    "destination": "<haproxy_ip>"
  }
]
```
  
- 보안 그룹을 생성한다.  

> $ cf create-security-group rabbitmq rule.json

```
Creating security group rabbitmq as admin...

OK		
```
  
- rabbitmq 서비스를 사용할수 있도록 생성한 보안 그룹을 적용한다.
> $ cf bind-running-security-group rabbitmq 
```
Binding security group rabbitmq to running as admin...
OK		
```
  
- 바인드가 적용되기 위해서 App을 재기동한다.

> $ cf restart rabbit-example-app 

```	
Restarting app rabbit-example-app in org system / space dev as admin...

Staging app and tracing logs...
   Downloading ruby_buildpack...
   Downloaded ruby_buildpack (5.2M)
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd creating container for instance 934daf45-8787-4d8a-86fd-bd6dbde78f30
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd successfully created container for instance 934daf45-8787-4d8a-86fd-bd6dbde7
   Downloading app package...
   Downloaded app package (3.2M)
   -----> Ruby Buildpack version 1.8.37


........
........
Instances starting...
Instances starting...

name:              rabbit-example-app
requested state:   started
routes:            rabbit-example-app.paasta.kr
last uploaded:     Mon 22 Nov 05:34:27 UTC 2021
stack:             cflinuxfs3
buildpacks:        
	name             version   detect output   buildpack name
	ruby_buildpack   1.8.37    ruby            ruby

type:           web
sidecars:       
instances:      1/1
memory usage:   1024M
     state     since                  cpu    memory   disk     details
#0   running   2021-11-22T05:34:36Z   0.0%   0 of 0   0 of 0   
```  


-  App이 정상적으로 RabbitMQ 서비스를 사용하는지 확인한다.


- 브라우저에서 확인

>![rabbitmq_image_12]

- 스토어 엔드포인트 테스트
```	
$ curl -XPOST -d 'test' https://rabbit-example-app.<YOUR-DOMAIN>/store -k  
$ curl -XGET https://rabbit-example-app.<YOUR-DOMAIN>/store -k  
test
```








[rabbitmq_image_01]:./images/rabbitmq/rabbitmq_image_01.png

[rabbitmq_image_12]:./images/rabbitmq/rabbitmq_image_12.png
[rabbitmq_image_13]:./images/rabbitmq/rabbitmq_image_13.png
[rabbitmq_image_14]:./images/rabbitmq/rabbitmq_image_14.png
[rabbitmq_image_15]:./images/rabbitmq/rabbitmq_image_15.png
[rabbitmq_image_16]:./images/rabbitmq/rabbitmq_image_16.png
[rabbitmq_image_17]:./images/rabbitmq/rabbitmq_image_17.png
[rabbitmq_image_18]:./images/rabbitmq/rabbitmq_image_18.png
[rabbitmq_image_19]:./images/rabbitmq/rabbitmq_image_19.png


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > RabbitMQ Service
