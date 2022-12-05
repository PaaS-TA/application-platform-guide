### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Redis Service

## Table of Contents  

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)  
  1.3. [참고자료](#1.3)  
  
2. [On-Demand-Redis 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)    
  2.6. [서비스 설치 확인](#2.6)  

3. [CF CLI를 이용한 On-Demand-Redis 서비스 브로커 등록](#3)  
  3.1. [PaaS-TA에서 서비스 신청](#3.1)  
  3.2. [Sample App 다운로드](#3.2)  
  3.3. [PaaS-TA에서 서비스 신청](#3.3)  
  3.4. [Sample App에 서비스 바인드 신청 및 App 확인](#3.4)  

4. [Portal을 이용한 Redis Service Test](#4)  
  4.1. [서비스 신청](#4.1)  



## <div id='1'> 1. 문서 개요

### <div id='1.1'> 1.1. 목적
본 문서(Redis 서비스팩 설치 가이드)는 PaaS-TA에서 제공되는 서비스팩인 Redis 서비스팩을 Bosh를 이용하여 설치하는 방법을 기술하였다.

### <div id='1.2'> 1.2. 범위
설치 범위는 Redis서비스팩을 검증하기 위한 기본 설치를 기준으로 작성하였다.

### <div id='1.3'> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  


## <div id='2'>  2. On-Demand Redis 서비스 설치
	
### <div id="2.1"/> 2.1. Prerequisite  

Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-bionic 1.122를 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version   OS             CPI  CID  
bosh-openstack-kvm-ubuntu-bionic-go_agent  1.122      ubuntu-bionic  -    ce507ae4-aca6-4a6d-b7c7-220e3f4aaa7d

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

- Service Deployment Git Repository URL : https://github.com/PaaS-TA/service-deployment/tree/v5.1.17

```
# Deployment 다운로드 파일 위치 경로 생성 및 설치 경로 이동
$ mkdir -p ~/workspace
$ cd ~/workspace

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/service-deployment.git -b v5.1.17

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
- redis에서 사용하는 변수는 bosh_url, bosh_client_admin_id, bosh_client_admin_secret, bosh_director_port, bosh_oauth_port, system_domain, paasta_admin_username, paasta_admin_password, bosh_version 이다.

> $ vi ~/workspace/common/common_vars.yml
```
... ((생략)) ...
bosh_url: "https://10.0.1.6"			# BOSH URL (e.g. "https://00.000.0.0")
bosh_client_admin_id: "admin"			# BOSH Client Admin ID
bosh_client_admin_secret: "ert7na4jpew48"	# BOSH Client Admin Secret('echo $(bosh int ~/workspace/paasta-deployment/bosh/{iaas}/creds.yml --path /admin_password)' 명령어를 통해 확인 가능)
bosh_director_port: 25555			# BOSH director port
bosh_oauth_port: 8443				# BOSH oauth port
bosh_version: 271.2				# BOSH version('bosh env' 명령어를 통해 확인 가능, on-demand service용, e.g. "271.2")
system_domain: "61.252.53.246.nip.io"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
paasta_admin_username: "admin"			# PaaS-TA Admin Username
paasta_admin_password: "admin"			# PaaS-TA Admin Password
... ((생략)) ...
```


- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/deployment/service-deployment/redis/vars.yml
```
# STEMCELL
stemcell_os: "ubuntu-bionic"                                      # Deployment Main Stemcell OS
stemcell_version: "1.122"                                        # Main Stemcell Version

# NETWORK
private_networks_name: "default"                                  # private network name

# MARIA DB
mariadb_azs: [z5]                                                 # mariadb azs
mariadb_instances: 1                                              # mariadb instances 
mariadb_vm_type: "medium"                                         # mariadb vm type
mariadb_persistent_disk_type: "2GB"                               # mariadb persistent disk type
mariadb_user_password: "admin"                                    # mariadb admin password
mariadb_port: 13306                                               # mariadb port (e.g. 13306) -- Do Not Use "3306"

# ON DEMAND BROKER
broker_azs: [z5]                                                  # broker azs
broker_instances: 1                                               # broker instances 
broker_vm_type: "service_medium"                                  # broker vm type

# REDIS
redis_azs: [z5]                                                   # redis azs
redis_vm_type: "medium"                                           # redis vm type
redis_persistent_disk_type: "1GB"                                 # redis persistent disk type

# PROPERTIES
broker_server_port: 8080                                          # broker server port

### On-Demand Dedicated Service Instance Properties ###
on_demand_service_instance_name: "redis"                          # On-Demand Service Instance Name
service_password: "PaaS-TA#2021!"                                 # On-Demand Redis Service password
service_port: 6379                                                # On-Demand Redis Service port

# SERVICE PLAN INFO
service_instance_guid: "54e2de61-de84-4b9c-afc3-88d08aadfcb6"            # Service Instance Guid
service_instance_name: "redis"                                           # Service Instance Name
service_instance_bullet_name: "Redis Dedicated Server Use"               # Service Instance bullet Name
service_instance_bullet_desc: "Redis Service Using a Dedicated Server"   # Service Instance bullet에 대한 설명을 입력
service_instance_plan_guid: "2a26b717-b8b5-489c-8ef1-02bcdc445720"       # Service Instance Plan Guid
service_instance_plan_name: "dedicated-vm"                               # Service Instance Plan Name
service_instance_plan_desc: "Redis service to provide a key-value store" # Service Instance Plan에 대한 설명을 입력
service_instance_org_limitation: "-1"                                    # Org에 설치할수 있는 Service Instance 개수를 제한한다. (-1일경우 제한없음)
service_instance_space_limitation: "-1"                                  # Space에 설치할수 있는 Service Instance 개수를 제한한다. (-1일경우 제한없음)
```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/service-deployment/redis/deploy.sh

```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"	# common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"		# bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d redis deploy --no-redact redis.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/service-deployment/redis  
$ sh ./deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e micro-bosh -d redis vms  

```
Using environment '10.0.1.6' as client 'admin'

Task 936167. Done

Deployment 'redis'

Instance                                                       Process State  AZ  IPs           VM CID                                   VM Type         Active  
mariadb/e35f3ece-9c34-41f4-a88e-d8365e9b8c70                   running        z5  10.30.255.25  vm-5168ec8d-f42f-40fa-9c3a-8635bf138b0a  medium          true  
paas-ta-on-demand-broker/13c11522-10dd-485c-bb86-3ac5337223d0  running        z5  10.30.255.26  vm-eab6e832-8b7c-49bc-ac04-80258896880d  service_medium  true  

2 vms

Succeeded
```

## <div id='3'> 3. CF CLI를 이용한 On-Demand-Redis 서비스 
### <div id='3.1'> 3.1. On-Demand-Redis 서비스 브로커 등록
Redis 서비스팩 배포가 완료 되었으면 Application에서 서비스 팩을 사용하기 위해서 먼저 On-Demand-Redis 서비스 브로커를 등록해 주어야 한다.
서비스 브로커 등록시에는 PaaS-TA에서 서비스 브로커를 등록할 수 있는 사용자로 로그인하여야 한다


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

	
- On-Demand-Redis 서비스 브로커를 등록한다.

> $ cf create-service-broker on-demand-redis-service admin cloudfoundry http://<paas-ta-on-demand-broker_ip>:8080 

```
$ cf create-service-broker on-demand-redis-service admin cloudfoundry http://10.30.255.26:8080
Creating service broker on-demand-redis-service as admin...
OK
```

- 등록된 On-Demand-Redis 서비스 브로커를 확인한다.

> $ cf service-brokers 
```
Getting service brokers as admin...

name                     url
on-demand-redis-service  http://10.30.255.26:8080
```


- 접근 가능한 서비스 목록을 확인한다.

> $ cf service-access 
```
Getting service access as admin...
broker: on-demand-redis-service
   offering   plan           access   orgs
   redis      dedicated-vm   none   

```
서비스 브로커 등록시 최초에는 접근을 허용하지 않는다. 따라서 access는 none으로 설정된다.


- 특정 조직에 해당 서비스 접근 허용을 할당하고 접근 서비스 목록을 다시 확인한다. (전체 조직)

> $ cf enable-service-access redis  <br>
```
Enabling access to all plans of service offering redis for all orgs as admin...
OK
```
> $ cf service-access 
```
Getting service access as admin...
	
broker: on-demand-redis-service
   offering   plan           access   orgs
   redis      dedicated-vm   all   
```

### <div id='3.2'> 3.2. Sample App 다운로드

- Sample App 묶음 다운로드
```
$ wget https://nextcloud.paas-ta.org/index.php/s/BoSbKrcXMmTztSa/download --content-disposition  
$ unzip paasta-service-samples-459dad9.zip  
$ cd paasta-service-samples/redis  
```

<br>

### <div id='3.3'> 3.3. PaaS-TA에서 서비스 신청
Sample App에서 Redis 서비스를 사용하기 위해서는 서비스 신청(Provision)을 해야 한다.
*참고: 서비스 신청시 PaaS-TA에서 서비스를 신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- 먼저 PaaS-TA Marketplace에서 서비스가 있는지 확인을 한다.

> $ cf marketplace

```
OK
service   plans          description
redis     dedicated-vm   A paasta source control service for application development.provision parameters : parameters {owner : owner}
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

> $ cf create-service redis dedicated-vm redis

```
Creating service instance redis in org system / space dev as admin...
OK

Create in progress. Use 'cf services' or 'cf service redis' to check operation status.
```


<br>

- 생성된 Redis 서비스 인스턴스의 status를 확인한다.
 * create in progress인 상태일경우 서비스 준비중이므로 서비스 이용 및 바인드, 삭제가 제한이되므로 create succeeded가 될때까지 기다려야 한다.
> $ cf service redis  

```
Showing info of service redis in org system / space dev as admin...

name:            redis
service:         redis
tags:            
plan:            dedicated-vm
description:     A paasta source control service for application development.provision parameters : parameters {owner : owner}
documentation:   https://paas-ta.kr
dashboard:       10.30.255.26

Showing status of last operation from service redis...

status:    create in progress
message:   
started:   2019-07-05T05:58:13Z
updated:   2019-07-05T05:58:16Z

There are no bound apps for this service.
```
- 생성된 Redis 서비스 인스턴스의 status가 create succeeded가 된것을 확인한다.
```
Showing info of service redis in org system / space dev as admin...

name:            redis
service:         redis
tags:            
plan:            dedicated-vm
description:     A paasta source control service for application development.provision parameters : parameters {owner : owner}
documentation:   https://paas-ta.kr
dashboard:       10.30.255.26

Showing status of last operation from service redis...

status:    create succeeded
message:   test
started:   2019-07-05T05:58:13Z
updated:   2019-07-05T06:01:20Z

There are no bound apps for this service.
```

<br>
	
- on-demand-service를 통해 서비스를 생성할 경우 해당 공간에 security-group 생성 및 자동적으로 할당이 된다.  
- Secuirty-group에 redis_[서비스 할당된 space guid] 가 생성된것을 확인한다.  
	
> $ cf space [space] --guid  
```
20bc9b52-c3d5-4cd2-94d9-7f444f9ab464
```

> $ cf security-groups  
```
Getting security groups as admin...
OK

     name                                         organization   space   lifecycle
#0   abacus                                       abacus-org     dev     running
#1   dns                                          <all>          <all>   running
     dns                                          <all>          <all>   staging
#2   public_networks                              <all>          <all>   running
     public_networks                              <all>          <all>   staging
#3   redis_20bc9b52-c3d5-4cd2-94d9-7f444f9ab464   system         dev     running
```


### <div id='3.4'> 3.4. Sample App에 서비스 바인드 신청 및 App 확인
서비스 신청이 완료되었으면 Sample App 에서는 생성된 서비스 인스턴스를 Bind 하여 App에서 Redis 서비스를 이용한다.
*참고: 서비스 Bind 신청시 PaaS-TA에서 서비스 Bind신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- manifest 파일을 확인한다.  

> $ vi manifest.yml   

```
---
applications:
- name: redis-example-app
  memory: 256M
  instances: 1
  path: .
  buildpacks: [ruby_buildpack]
```

- --no-start 옵션으로 App을 배포한다.

> $ cf push --no-start 
```  
Pushing app redis-example-app to org system / space dev as admin...
Applying manifest file /home/ubuntu/workspace/samples/paasta-service-samples/redis/manifest.yml...
Manifest applied
Packaging files to upload...
Uploading files...
 1.23 MiB / 1.23 MiB [===================================================================================================

Waiting for API to complete processing files...

name:              redis-example-app
requested state:   stopped
routes:            redis-example-app.paastacloud.shop
last uploaded:     
stack:             
buildpacks:        

type:           web
sidecars:       
instances:      0/1
memory usage:   256M
     state   since                  cpu    memory   disk     details
#0   down    2021-11-22T05:39:06Z   0.0%   0 of 0   0 of 0   
```  
  
- Sample Web App에서 생성한 서비스 인스턴스 바인드 신청을 한다.

> $ cf bind-service redis-example-app redis 

```	
Binding service redis to app redis-example-app in org system / space dev as admin...
OK
```
	
- 바인드가 적용되기 위해서 App을 재기동한다.

> $ cf restart redis-example-app 

```	
Restarting app redis-example-app in org system / space dev as admin...

Staging app and tracing logs...
   Downloading ruby_buildpack...
   Downloaded ruby_buildpack
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd creating container for instance 4a47d02a-24d6-4046-b2b8-866b915eaf6a
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd successfully created container for instance 4a47d02a-24d6-4046-b2b8-866b915e
   Downloading app package...
   Downloaded app package (1.4M)
   -----> Ruby Buildpack version 1.8.37

........
........
Instances starting...
Instances starting...

name:              redis-example-app
requested state:   started
routes:            redis-example-app.paastacloud.shop
last uploaded:     Mon 22 Nov 05:40:50 UTC 2021
stack:             cflinuxfs3
buildpacks:        
	name             version   detect output   buildpack name
	ruby_buildpack   1.8.37    ruby            ruby

type:           web
sidecars:       
instances:      1/1
memory usage:   256M
     state     since                  cpu    memory   disk     details
#0   running   2021-11-22T05:41:01Z   0.0%   0 of 0   0 of 0  
```  


<br>

App이 정상적으로 Redis 서비스를 사용하는지 확인한다.

- curl 로 확인

```
$ export APP=redis-example-app.[CF Domain]
$ curl -X PUT $APP/foo -d 'data=bar'
success
$ curl -X GET $APP/foo
bar
$ curl -X DELETE $APP/foo
success
```


<br>

## <div id='4'> 4. Portal을 이용한 Redis Service Test
사용자 및 관리자 포탈이 설치가 되어있으면 포탈을 통해서 레디스 서비스 신청 및 바인드, 테스트가 가능하다.


- 관리자 포탈에 접속해 서비스 관리의 서비스 브로커 페이지에서 브로커 리스트를 확인한다..
![1]
- On-Demand-Redis 서비스 브로커를 등록한다.
![2]
![3]
- 등록된 On-Demand-Redis 서비스 브로커를 확인한다.
![4]
- 서비스관리의 서비스 제어 페이지에서 접근 가능한 서비스 목록을 확인한다.
![5]

서비스 브로커 등록시 최초에는 접근을 허용하지 않는다. 따라서 access는 none으로 설정된다.
- 특정 조직에 해당 서비스 접근 허용을 할당하고 접근 서비스 목록을 다시 확인한다. (전체 조직)
![6]

### <div id='4.1'> 4.1. 서비스 신청
사용자 포탈에서 서비스 신청하기 위해서는 관리자 포탈의 카탈로그페이지에서 서비스 등록을 먼저 해주어야 사용이 가능하다.

- 관리자 포탈의 운영관리의 카탈로그 페이지로 이동해 서비스 등록을 한다.
![7]
앱 바인드 파라미터는 app_guid 자동입력을 추가, 온디멘드 Y 로 설정후 서비스 등록을 진행한다.
- 사용자 포탈 로그인 후 카탈로그 페이지에서 서비스를 생성한다.
![8]


- 생성된 Redis 서비스 인스턴스의 status를 확인한다.
Service status : in progress
![9]
 
Service status : created succeed
![10]

- 관리자포탈 보안의 시큐리티그룹 페이지로 이동해 redis_[서비스 할당된 space guid] 가 생성된것을 확인한다.
![11]


[redis_image_01]:./images/redis/redis_image_01.png
[redis_image_02]:./images/redis/redis_image_02.png
[redis_image_03]:./images/redis/redis_image_03.png
[redis_image_04]:./images/redis/redis_image_04.png
[redis_image_05]:./images/redis/redis_image_05.png
[redis_image_06]:./images/redis/redis_image_06.png
[redis_image_07]:./images/redis/redis_image_07.png
[redis_image_08]:./images/redis/redis_image_08.png
[redis_image_09]:./images/redis/redis_image_09.png
[redis_image_10]:./images/redis/redis_image_10.png
[redis_image_11]:./images/redis/redis_image_11.png
[redis_image_12]:./images/redis/redis_image_12.png
[redis_image_13]:./images/redis/redis_image_13.png
[redis_image_14]:./images/redis/redis_image_14.png
[redis_image_15]:./images/redis/redis_image_15.png
[redis_image_16]:./images/redis/redis_image_16.png
[redis_image_17]:./images/redis/redis_image_17.png
[redis_image_18]:./images/redis/redis_image_18.png
[redis_image_19]:./images/redis/redis_image_19.png
[redis_image_20]:./images/redis/redis_image_20.png
[redis_image_21]:./images/redis/redis_image_21.png
[redis_image_22]:./images/redis/redis_image_22.png
[redis_image_23]:./images/redis/redis_image_23.png
[redis_image_24]:./images/redis/redis_image_24.png
[redis_image_25]:./images/redis/redis_image_25.png
[redis_image_26]:./images/redis/redis_image_26.png
[1]:./images/redis/redis_test1.PNG
[2]:./images/redis/redis_test2.PNG
[3]:./images/redis/redis_test3.PNG
[4]:./images/redis/redis_test4.PNG
[5]:./images/redis/redis_test5.PNG
[6]:./images/redis/redis_test6.PNG
[7]:./images/redis/redis_test7.PNG
[8]:./images/redis/redis_test8.PNG
[9]:./images/redis/redis_test9.PNG
[10]:./images/redis/redis_test10.PNG
[11]:./images/redis/redis_test11.PNG



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Redis Service
