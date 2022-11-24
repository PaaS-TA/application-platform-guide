### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > GlusterFS Service

## Table of Contents  

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)  
  1.3. [참고자료](#1.3)  
  
2. [GlusterFS 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)    
  2.6. [서비스 설치 확인](#2.6)  
  
3. [GlusterFS 연동 Sample App 설명](#3)    
  3.1. [서비스 브로커 등록](#3.1)   
  3.2. [Sample App 다운로드](#3.2)    
  3.3. [PaaS-TA에서 서비스 신청](#3.3)   
  3.4. [Sample App에 서비스 바인드 신청 및 App 확인](#3.4)   


## <div id="1"/> 1. 문서 개요

### <div id="1.1"/>1.1. 목적
본 문서(GlusterFS 서비스팩 설치 가이드)는 PaaS-TA에서 제공되는 서비스팩인 GlusterFS 서비스팩을 Bosh를 이용하여 설치 하는 방법을 기술하였다.  

### <div id="1.2"/> 1.2. 범위
설치 범위는 GlusterFS 서비스팩을 검증하기 위한 기본 설치를 기준으로 작성하였다.


### <div id="1.3"/>1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

## <div id="2"/>2. GlusterFS 서비스 설치

### <div id="2.1"/> 2.1. Prerequisite  

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.  
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.  
서비스 브로커와 연결 할 Swift & GlusterFS Server가 설치되있어야 한다.

### <div id="2.2"/> 2.2. Stemcell 확인

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
- glusterfs에서 사용하는 변수는 system_domain, paasta_admin_username, paasta_admin_password 이다.

> $ vi ~/workspace/common/common_vars.yml
```
... ((생략)) ...

system_domain: "61.252.53.246.nip.io"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
paasta_admin_username: "admin"			# PaaS-TA Admin Username
paasta_admin_password: "admin"			# PaaS-TA Admin Password

... ((생략)) ...

```



- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/service-deployment/glusterfs/vars.yml

```
# STEMCELL
stemcell_os: "ubuntu-bionic"                                     # stemcell os
stemcell_version: "1.122"                                       # stemcell version


# NETWORK
private_networks_name: "default"                                 # private network name
public_networks_name: "vip"                                      # public network name


# MYSQL
mysql_azs: [z4]                                                  # mysql azs
mysql_instances: 1                                               # mysql instances 
mysql_vm_type: "medium"                                          # mysql vm type
mysql_persistent_disk_type: "2GB"                                # mysql persistent disk type
mysql_port: 13306                                                # mysql port (e.g. 13306) -- Do Not Use "3306"
mysql_admin_username: "<MYSQL_ADMIN_USERNAME>"                   # mysql admin username (e.g. "root")
mysql_admin_password: "<MYSQL_ADMIN_PASSWORD>"                   # mysql admin password (e.g. "admin#1234" 영어/숫자/특수문자 혼용 8자리 이상 또는 2종류 혼용 10자리 이상)


# GLUSTERFS SERVER
glusterfs_url: "<GLUSTERFS_PUBLIC_IP>"                           # Glusterfs 서비스 public 주소
glusterfs_tenantname: "<GLUSTERFS_TENANT_NAME>"                  # Glusterfs 서비스 테넌트 이름(e.g. "service")
glusterfs_username: "<GLUSTERFS_USERNAME>"                       # Glusterfs 서비스 계정 아이디(e.g. "swift")
glusterfs_password: "<GLUSTERFS_PASSWORD>"                       # Glusterfs 서비스 암호(e.g. "password")
glusterfs_domainname: "<GLUSTERFS_DOMAIN_NAME>"                  # Glusterfs 서비스 도메인 이름 (e.g. "default")
swiftproxy_port: "<SWIFT_PROXY_PORT>"                            # Glusterfs 서비스 swift proxy port (e.g. "10008")
auth_port: "<AUTH_PORT>"                                         # Glusterfs 서비스 auth port (e.g. "15001")

# GLUSTERFS_BROKER
broker_azs: [z4]                                                 # glusterfs broker azs
broker_instances: 1                                              # glusterfs broker instances 
broker_persistent_disk_type: "4GB"                               # glusterfs broker persistent disk type
broker_vm_type: "small"                                          # glusterfs broker vm type


# GLUSTERFS_BROKER_REGISTRAR
broker_registrar_azs: [z4]                                       # broker registrar azs
broker_registrar_instances: 1                                    # broker registrar instances 
broker_registrar_vm_type: "small"                                # broker registrar vm type


# GLUSTERFS_BROKER_DEREGISTRAR
broker_deregistrar_azs: [z4]                                     # broker deregistrar azs
broker_deregistrar_instances: 1                                  # broker deregistrar instances 
broker_deregistrar_vm_type: "small"                              # broker deregistrar vm type
```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/service-deployment/glusterfs/deploy.sh

```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"    # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"        # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

bosh -e ${BOSH_ENVIRONMENT} -n -d glusterfs deploy --no-redact glusterfs.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/service-deployment/glusterfs  
$ sh ./deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e micro-bosh -d glusterfs vms  

```
Using environment '10.0.1.6' as client 'admin'

Task 1343. Done

Deployment 'glusterfs'

Instance                                                      Process State  AZ  IPs          VM CID                                   VM Type  Active  
mysql/8770bc70-8681-4079-8360-086219d6231b                    running        z3  10.30.52.10  vm-96697221-0ff9-4520-8a68-2314c62057a5  medium   true  
paasta-glusterfs-broker/229fb890-645b-4213-89a1-fc2116de3f54  running        z3  10.30.52.11  vm-ace55b8f-3ce0-4482-b03b-96fbc567592e  medium   true  

2 vms

Succeeded
```

## <div id="3"/>3. GlusterFS 연동 Sample App 설명
본 Sample Web App은 PaaS-TA에 배포되며 GlusterFS의 서비스를 Provision과 Bind를 한 상태에서 사용이 가능하다.

### <div id="3.1"/> 3.1. 서비스 브로커 등록  

GlusterFS 서비스팩 배포가 완료 되었으면 Application에서 서비스 팩을 사용하기 위해서 먼저 GlusterFS 서비스 브로커를 등록해 주어야 한다.
서비스 브로커 등록시에는 PaaS-TA에서 서비스 브로커를 등록할 수 있는 사용자로 로그인 하여야 한다

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

- GlusterFS 서비스 브로커를 등록한다.  
  
> $ cf create-service-broker glusterfs-service admin cloudfoundry http://<paasta-glusterfs-broker_ip>:8080
```  
$ cf create-service-broker glusterfs-service admin cloudfoundry http://10.30.52.11:8080
Creating service broker glusterfs-service as admin...
OK
```  

- 등록된 GlusterFS 서비스 브로커를 확인한다.

> $ cf service-brokers   
```  
Getting service brokers as admin...

name                           url
glusterfs-service              http://10.30.52.11:8080
```  

- 접근 가능한 서비스 목록을 확인한다.

>`$ cf service-access`  
```  
Getting service access as admin...
broker: glusterfs-service
   service     plan               access   orgs
   glusterfs   glusterfs-5Mb      none
   glusterfs   glusterfs-100Mb    none
   glusterfs   glusterfs-1000Mb   none
```  
>서비스 브로커 등록시 최초에는 접근을 허용하지 않는다. 따라서 access는 none으로 설정된다.

- 특정 조직에 해당 서비스 접근 허용을 할당하고 접근 서비스 목록을 다시 확인한다. (전체 조직)

> $ cf enable-service-access glusterfs   
```
Enabling access to all plans of service glusterfs for all orgs as admin...
OK
```
> $ cf service-access   
```  
Getting service access as admin...
broker: glusterfs-service
   service     plan               access   orgs
   glusterfs   glusterfs-5Mb      all
   glusterfs   glusterfs-100Mb    all
   glusterfs   glusterfs-1000Mb   all
```  


### <div id='3.2'> 3.2. Sample App 다운로드

- Sample App 묶음 다운로드
```
$ wget https://nextcloud.paas-ta.org/index.php/s/BoSbKrcXMmTztSa/download --content-disposition  
$ unzip paasta-service-samples-459dad9.zip  
$ cd paasta-service-samples/glusterfs  
```

<br>

### <div id="3.3"/> 3.3. PaaS-TA에서 서비스 신청
Sample App에서 GlusterFS 서비스를 사용하기 위해서는 서비스 신청(Provision)을 해야 한다.
*참고: 서비스 신청시 PaaS-TA에서 서비스를 신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.

- 먼저 PaaS-TA Marketplace에서 서비스가 있는지 확인을 한다.

> $ cf marketplace

```  
Getting services from marketplace in org system / space dev as admin...
OK

service      plans                                              description
glusterfs    glusterfs-5Mb, glusterfs-100Mb, glusterfs-1000Mb   A simple glusterfs implementation 

TIP:  Use 'cf marketplace -s SERVICE' to view descriptions of individual plans of a given service.
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

> $ cf create-service glusterfs glusterfs-1000Mb glusterfs-service-instance 
```  
Creating service instance glusterfs-service-instance in org system / space dev as admin...
OK
```  

<br>


- 생성된 GlusterFS 서비스 인스턴스를 확인한다.

> $ cf services
```  
Getting services in org system / space dev as admin...
OK

name                        service     plan                 bound apps            last operation
glusterfs-service-instance  glusterfs   glusterfs-1000Mb                           create succeeded
```  

<br>


### <div id='3.4'> 3.4. Sample App에 서비스 바인드 신청 및 App 확인  

서비스 신청이 완료되었으면 Sample Web App 에서는 생성된 서비스 인스턴스를 Bind 하여 App에서 GlusterFS 서비스를 이용한다.
*참고: 서비스 Bind 신청시 개방형 클라우드 플랫폼에서 서비스 Bind신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.
  
- manifest 파일을 확인한다. (swift_region을 GlusterFS 서버의 사용하려는 region으로 설정한다.)  

> $ vi manifest.yml   
```
---
applications:
- name: hello-spring-glusterfs
  memory: 1G
  instances: 1
  path: hello-spring-glusterfs.war
  buildpacks:
  - java_buildpack
  env:
    swift_region: paasta
```

- --no-start 옵션으로 App을 배포한다.

> $ cf push --no-start 
```  
Applying manifest file /home/ubuntu/workspace/samples/paasta-service-samples/gluserfs/manifest.yml...
Manifest applied
Packaging files to upload...
Uploading files...
 17.06 MiB / 17.06 MiB [=================================================================================================

Waiting for API to complete processing files...

name:              hello-spring-glusterfs
requested state:   stopped
routes:            hello-spring-glusterfs.paasta.kr
last uploaded:     
stack:             
buildpacks:        

type:           web
sidecars:       
instances:      0/1
memory usage:   1024M
     state   since                  cpu    memory   disk     details
#0   down    2021-11-22T05:13:12Z   0.0%   0 of 0   0 of 0   
```  
  
- Sample Web App에서 생성한 서비스 인스턴스 바인드 신청을 한다.

> $ cf bind-service hello-spring-glusterfs glusterfs-service-instance

```
Binding service glusterfs-service-instance to app hello-spring-glusterfs in org system / space dev as admin...
OK
```

App 구동 시 Service와의 통신을 위하여 보안 그룹을 추가한다.

- rule.json을 편집한다.  
> $ vi rule.json   
```
## glusterfs의 IP와 PORT(swiftproxy_port, auth_port)를 destination에 설정
[
  {
    "protocol": "tcp",
    "ports": "<auth_port>, <swiftproxy_port>",
    "destination": "<glusterfs_ip>",
    "log": true,
    "description": "Allow tcp traffic to gluster"
  }
]
```
  
- 보안 그룹을 생성한다.  

> $ cf create-security-group glusterfs rule.json  

```
Creating security group glusterfs as admin...

OK		
```
  
- GlusterFS 서비스를 사용할수 있도록 생성한 보안 그룹을 적용한다.
> $ cf bind-running-security-group glusterfs  
```
Binding security group glusterfs to running as admin...
OK		
```
  
- 바인드가 적용되기 위해서 App을 재기동한다.

> $ cf restart hello-spring-glusterfs 

```	
Restarting app hello-spring-glusterfs in org system / space dev as admin...

Staging app and tracing logs...
   Downloading java_buildpack...
   Downloaded java_buildpack
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd creating container for instance 678aa272-945b-41a9-8924-0782891d0cc4
   Cell 4a88ce8b-1e72-485a-8f62-1fe0c6b9a7cd successfully created container for instance 678aa272-945b-41a9-8924-0782891d0cc4
   Downloading app package...
   Downloaded app package (30.5M)

........
........
Instances starting...
Instances starting...

name:              hello-spring-glusterfs
requested state:   started
routes:            hello-spring-glusterfs.paasta.kr
last uploaded:     Mon 22 Nov 05:19:59 UTC 2021
stack:             cflinuxfs3
buildpacks:        
	name             version                                                             detect output   buildpack na
	java_buildpack   v4.37-https://github.com/cloudfoundry/java-buildpack.git#ab2b4512   java            java

type:           web
sidecars:       
instances:      1/1
memory usage:   1024M
     state     since                  cpu    memory    disk       details
#0   running   2021-11-22T05:20:19Z   0.0%   0 of 1G   8K of 1G   

```  


- App이 정상적으로 GlusterFS 서비스를 사용하는지 확인한다.  
	
![glusterfs_image_18]
<br>

![glusterfs_image_19]

[glusterfs_image_01]:./images/glusterfs/glusterfs_image_01.png

[glusterfs_image_07]:./images/glusterfs/glusterfs_image_07.png
[glusterfs_image_08]:./images/glusterfs/glusterfs_image_08.png
[glusterfs_image_09]:./images/glusterfs/glusterfs_image_09.png
[glusterfs_image_10]:./images/glusterfs/glusterfs_image_10.png
[glusterfs_image_11]:./images/glusterfs/glusterfs_image_11.png
[glusterfs_image_12]:./images/glusterfs/glusterfs_image_12.png
[glusterfs_image_13]:./images/glusterfs/glusterfs_image_13.png
[glusterfs_image_14]:./images/glusterfs/glusterfs_image_14.png
[glusterfs_image_15]:./images/glusterfs/glusterfs_image_15.png
[glusterfs_image_16]:./images/glusterfs/glusterfs_image_16.png
[glusterfs_image_17]:./images/glusterfs/glusterfs_image_17.jpeg
[glusterfs_image_18]:./images/glusterfs/glusterfs_image_18.png
[glusterfs_image_19]:./images/glusterfs/glusterfs_image_19.png


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > GlusterFS Service
