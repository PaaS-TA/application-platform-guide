### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > MySQL Service

## Table of Contents  

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)  
  1.3. [참고자료](#1.3)  
  
2. [MySQL 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)    
  2.6. [서비스 설치 확인](#2.6)   
  
3. [MySQL 연동 Sample Web App 설명](#3)  
  3.1. [서비스 브로커 등록](#3.1)  
  3.2. [Sample Web App 다운로드](#3.2)  
  3.3. [PaaS-TA에서 서비스 신청](#3.3)  
  3.4. [Sample Web App 배포 및 MySQL바인드 확인](#3.4)  

4. [MySQL Client 툴 접속](#4)  
  4.1. [HeidiSQL 설치 및 연결](#4.1)  






## <div id='1'> 1. 문서 개요
### <div id='1.1'> 1.1. 목적

본 문서(MySQL 서비스팩 설치 가이드)는 PaaS-TA에서 제공되는 서비스팩인 MySQL 서비스팩을 Bosh를 이용하여 설치 하는 방법을 기술하였다.
	
	
### <div id='1.2'> 1.2. 범위
설치 범위는 MySQL 서비스팩을 검증하기 위한 기본 설치를 기준으로 작성하였다.

### <div id='1.3'> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

## <div id='2'> 2. MySQL 서비스 설치

### <div id="2.1"/> 2.1. Prerequisite  

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.  
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.  

### <div id="2.2"/> 2.2. Stemcell 확인

Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-bionic 1.171를 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version   OS             CPI  CID  
bosh-openstack-kvm-ubuntu-bionic-go_agent  1.171      ubuntu-bionic  -    ce507ae4-aca6-4a6d-b7c7-220e3f4aaa7d

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

- Service Deployment Git Repository URL : https://github.com/PaaS-TA/service-deployment/tree/v5.1.20

```
# Deployment 다운로드 파일 위치 경로 생성 및 설치 경로 이동
$ mkdir -p ~/workspace
$ cd ~/workspace

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/service-deployment.git -b v5.1.20
```

### <div id="2.4"/> 2.4. Deployment 파일 수정

BOSH Deployment manifest는 Components 요소 및 배포의 속성을 정의한 YAML 파일이다.  
Deployment 파일에서 사용하는 network, vm_type, disk_type 등은 Cloud config를 활용하고, 활용 방법은 PaaS-TA AP 설치 가이드를 참고한다.  

- Cloud config 설정 내용을 확인한다.   

> $ bosh -e ${BOSH_ENVIRONMENT} cloud-config   

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

> $ vi ~/workspace/service-deployment/mysql/vars.yml	
```
# STEMCELL
stemcell_os: "ubuntu-bionic"                                     # stemcell os
stemcell_version: "1.171"                                       # stemcell version

# NETWORK
private_networks_name: "default"                                 # private network name

# MYSQL
mysql_azs: [z4]                                                  # mysql azs
mysql_instances: 3                                               # mysql instances (N)
mysql_vm_type: "small"                                           # mysql vm type
mysql_persistent_disk_type: "8GB"                                # mysql persistent disk type
mysql_port: 13306                                                # mysql port (e.g. 13306) -- Do Not Use "3306"
mysql_admin_password: "<MYSQL_ADMIN_PASSWORD>"                   # mysql admin password (e.g. "admin!Service")

# PROXY
proxy_azs: [z4]                                                  # proxy azs
proxy_vm_type: "small"                                           # proxy vm type
proxy_mysql_port: 13307                                          # proxy mysql port (e.g. 13307) -- Do Not Use "3306"
proxy_static_ip: "<PROXY_STATIC_IP>"                             # proxy ip (e.g. "10.0.161.100")

# MYSQL_BROKER
mysql_broker_azs: [z4]                                           # mysql broker azs
mysql_broker_instances: 1                                        # mysql broker instances (1)
mysql_broker_vm_type: "small"                                    # mysql broker vm type
mysql_broker_services_plan_a_name: "<MYSQL_BROKER_SERVICE_PLAN_A_NAME>"   # mysql broker service small plan name (e.g. "Mysql-Plan1-10con")
mysql_broker_services_plan_a_connection: 10                      # mysql broker service small plan user connections
mysql_broker_services_plan_b_name: "<MYSQL_BROKER_SERVICE_PLAN_B_NAME>"   # mysql broker service big plan name (e.g. "Mysql-Plan2-100con")
mysql_broker_services_plan_b_connection: 100                     # mysql broker service big plan user connections
```


### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)


> $ vi ~/workspace/service-deployment/mysql/deploy.sh

```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"    # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"        # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d mysql deploy --no-redact mysql.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/service-deployment/mysql  
$ sh ./deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} -d mysql vms  

```
Using environment '10.0.1.6' as client 'admin'

Task 4525. Done

Deployment 'mysql'

Instance                                                       Process State  AZ  IPs            VM CID                                   VM Type  Active  
mysql-broker/0150c7f3-8920-45e6-839b-29884dc61301              running        z5  10.30.107.165  vm-214663a8-fcbc-4ae4-9aae-92027b9725a9  minimal  true  
mysql/00e8731f-5b13-421e-b633-0813a33db476                     running        z5  10.30.107.167  vm-7c3edc00-3074-4e98-9c89-9e9ba83b47e4  minimal  true  
mysql/00e8731f-5b13-421e-b633-0813a33db476                     running        z5  10.30.107.164  vm-81ecdc43-03d2-44f5-9b89-c6cdaa443d8b  minimal  true  
mysql/00e8731f-5b13-421e-b633-0813a33db476                     running        z5  10.30.107.168  vm-e447eb75-1119-451f-adc9-71b0a6ef1a6a  minimal  true  
proxy/2adc060d-a30b-46bc-b5f7-a4c09db1b189                     running        z5  10.30.107.160  vm-e447eb75-1119-451f-adc9-71b0a6ef1a6a  minimal  true  

5 vms

Succeeded
```	

## <div id='3'> 3. MySQL 연동 Sample Web App 설명  

본 Sample App은 MySQL의 서비스를 Provision한 상태에서 PaaS-TA에 배포하면 MySQL서비스와 bind되어 사용할 수 있다.  

### <div id='3.1'> 3.1. MySQL 서비스 브로커 등록  
Mysql 서비스팩 배포가 완료 되었으면 Application에서 서비스 팩을 사용하기 위해서 먼저 MySQL 서비스 브로커를 등록해 주어야 한다.  
서비스 브로커 등록시 PaaS-TA에서 서비스브로커를 등록할 수 있는 사용자로 로그인이 되어 있어야 한다.

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
	
- MySQL 서비스 브로커를 등록한다.

>`$ cf create-service-broker mysql-service-broker admin cloudfoundry http://<mysql-broker_ip>:8080`
```  
$ cf create-service-broker mysql-service-broker admin cloudfoundry http://10.30.107.167:8080
Creating service broker mysql-service-broker as admin...
OK
```  

- 등록된 MySQL 서비스 브로커를 확인한다.

>`$ cf service-brokers`
```  
$ cf service-brokers
Getting service brokers as admin...

name                      url
mysql-service-broker      http://10.30.107.167:8080
```  

- 접근 가능한 서비스 목록을 확인한다.

>`$ cf service-access`
```  
$ cf service-access
Getting service access as admin...
broker: mysql-service-broker
   service    plan                 access   orgs
   Mysql-DB   Mysql-Plan1-10con    none
   Mysql-DB   Mysql-Plan2-100con   none
```  
서비스 브로커 생성시 디폴트로 접근을 허용하지 않는다.

- 특정 조직에 해당 서비스 접근 허용을 할당하고 접근 서비스 목록을 다시 확인한다. (전체 조직)

> $ cf enable-service-access Mysql-DB  
```
Enabling access to all plans of service Mysql-DB for all orgs as admin...
OK
```
> $ cf service-access   
```
Getting service access as admin...
broker: mysql-service-broker
   service    plan                 access   orgs
   Mysql-DB   Mysql-Plan1-10con    all
   Mysql-DB   Mysql-Plan2-100con   all
```  

### <div id='3.2'> 3.2. Sample Web App 다운로드  

Sample App은 PaaS-TA에 App으로 배포되며 App구동시 Bind 된 MySQL 서비스 연결 정보로 접속하여 초기 데이터를 생성하게 된다.    
브라우져를 통해 App에 접속 후 "MYSQL 데이터 가져오기"를 통해 초기 생성된 데이터를 조회 할 수 있다.  

- Sample App 묶음 다운로드
```
$ wget https://nextcloud.paas-ta.org/index.php/s/BoSbKrcXMmTztSa/download --content-disposition  
$ unzip paasta-service-samples-459dad9.zip  
$ cd paasta-service-samples/mysql  
```

### <div id='3.3'> 3.3. PaaS-TA에서 서비스 신청  
Sample App에서 MySQL 서비스를 사용하기 위해서는 서비스 신청(Provision)을 해야 한다.  

*참고: 서비스 신청시 PaaS-TA에서 서비스를 신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.  

- 먼저 PaaS-TA Marketplace에서 서비스가 있는지 확인을 한다.  

> $ cf marketplace   
```  
Getting services from marketplace in org org system / space dev as admin...
OK

service      plans                                    description
Mysql-DB     Mysql-Plan1-10con, Mysql-Plan2-100con*   A simple mysql implementation

TIP:  Use 'cf marketplace -s SERVICE' to view descriptions of individual plans of a given service.
```  

- 서비스 인스턴스 신청 명령어
```
cf create-service [SERVICE] [PLAN] [SERVICE_INSTANCE]

[SERVICE] : Marketplace에서 보여지는 서비스 명
[PLAN] : 서비스에 대한 정책
[SERVICE_INSTANCE] : 생성할 서비스 인스턴스 이름
```
	
- Marketplace에서 원하는 서비스가 있으면 서비스 신청(Provision)을 한다.  

> $ cf create-service Mysql-DB Mysql-Plan2-100con mysql-service-instance   
```  
Creating service instance mysql-service-instance in org org system / space dev as admin...
OK

Attention: The plan `Mysql-Plan2-100con` of service `Mysql-DB` is not free.  The instance `mysql-service-instance` will incur a cost.  Contact your administrator if you think this is in error.
```  

- 생성된 MySQL 서비스 인스턴스를 확인한다.  

> $ cf services 
```  
Getting services in org system / space dev as admin...
OK

name                      service    plan                 bound apps            last operation
mysql-service-instance    Mysql-DB   Mysql-Plan2-100con                         create succeeded
```  

### <div id='3.4'> 3.4. Sample Web App 배포 및 MySQL바인드 확인   
서비스 신청이 완료되었으면 Sample Web App 에서는 생성된 서비스 인스턴스를 Bind 하여 App에서 MySQL 서비스를 이용한다.  
*참고: 서비스 Bind 신청시 PaaS-TA에서 서비스 Bind신청 할 수 있는 사용자로 로그인이 되어 있어야 한다.  

- manifest 파일을 확인한다.  

> $ vi manifest.yml   

```
---
applications:
- name: mysql-sample-app
  memory: 1024M
  instances: 1
  buildpack: java_buildpack
  path: mysql-sample-app.war
  env:
    mysql_datasource_driver-class-name: com.mysql.cj.jdbc.Driver
    mysql_datasource_jdbc-url: jdbc:\${vcap.services.mysql-service-instance.credentials.uri}
    mysql_datasource_username: \${vcap.services.mysql-service-instance.credentials.username}
    mysql_datasource_password: \${vcap.services.mysql-service-instance.credentials.password}

```

- --no-start 옵션으로 App을 배포한다.  
> $ cf push --no-start  
```  
Applying manifest file /home/ubuntu/workspace/samples/paasta-service-samples/mysql/manifest.yml...
Manifest applied
Packaging files to upload...
Uploading files...
 26.48 MiB / 26.48 MiB [================================================================================================================] 100.00% 1s

Waiting for API to complete processing files...

name:              mysql-sample-app
requested state:   stopped
routes:            mysql-sample-app.paasta.kr
last uploaded:     
stack:             
buildpacks:        

type:           web
sidecars:       
instances:      0/1
memory usage:   1024M
     state   since                  cpu    memory   disk     details
#0   down    2021-11-22T05:21:57Z   0.0%   0 of 0   0 of 0   
```  
	
- Sample Web App에서 생성한 서비스 인스턴스 바인드 신청을 한다.

> $ cf bind-service mysql-sample-app mysql-service-instance  

```
Binding service mysql-service-instance to app mysql-sample-app in org system / space dev as admin...
OK
```

App 구동 시 Service와의 통신을 위하여 보안 그룹을 추가한다.

- rule.json을 편집한다.  
> $ vi rule.json   
```
## mysql의 proxy IP를 destination에 설정
[
  {
    "protocol": "tcp",
    "destination": "<proxy_ip>",
    "ports": "13307"
  }
]
```
<br>

- 보안 그룹을 생성한다.  

> $ cf create-security-group mysql rule.json  

```
Creating security group mysql as admin...

OK		
```

<br>

- Mysql 서비스를 사용할수 있도록 생성한 보안 그룹을 적용한다.
> $ cf bind-running-security-group mysql  
```
Binding security group mysql to running as admin...
OK		
```
	
- App을 재기동 한다.  


> $ cf restart mysql-sample-app  
```	
Restarting app mysql-sample-app in org system / space dev as admin...

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

name:              mysql-sample-app
requested state:   started
routes:            mysql-sample-app.paasta.kr
last uploaded:     Mon 22 Nov 05:23:48 UTC 2021
stack:             cflinuxfs3
buildpacks:        
	name             version                                                             detect output   buildpack name
	java_buildpack   v4.37-https://github.com/cloudfoundry/java-buildpack.git#ab2b4512   java            java
```  

- App이 정상적으로 MySQL 서비스를 사용하는지 확인한다.  

![update_mysql_vsphere_34]  

## <div id='4'> 4. MySQL Client 툴 접속  

Application에 바인딩 된 MySQL 서비스 연결정보는 Private IP로 구성되어 있기 때문에 MySQL Client 툴에서 직접 연결할수 없다. 따라서 MySQL Client 툴에서 SSH 터널, Proxy 터널 등을 제공하는 툴을 사용해서 연결하여야 한다. 본 가이드는 SSH 터널을 이용하여 연결 하는 방법을 제공하며 MySQL Client 툴로써는 오픈 소스인 HeidiSQL로 가이드한다. HeidiSQL 에서 접속하기 위해서 먼저 SSH 터널링 할수 있는 VM 인스턴스를 생성해야한다. 이 인스턴스는 SSH로 접속이 가능해야 하고 접속 후 Open PaaS 에 설치한 서비스팩에 Private IP 와 해당 포트로 접근이 가능하도록 시큐리티 그룹을 구성해야 한다. 이 부분은 vSphere관리자 및 OpenPaaS 운영자에게 문의하여 구성한다.  

### <div id='4.1'> 4.1. HeidiSQL 설치 및 연결  

HeidiSQL 프로그램은 무료로 사용할 수 있는 오픈소스 소프트웨어이다.  

- HeidiSQL을 다운로드 하기 위해 아래 URL로 이동하여 설치파일을 다운로드 한다.  

>[http://www.heidisql.com/download.php](http://www.heidisql.com/download.php)

>![mysql_vsphere_4.1.01]

<br>

- 다운로드한 설치파일을 실행한다.

>![mysql_vsphere_4.1.02]

<br>

- HeidSQL 설치를 위한 안내사항이다. Next 버튼을 클릭한다.

>![mysql_vsphere_4.1.03]

<br>

- 프로그램 라이선스에 관련된 내용이다. 동의(I accept the agreement)에 체크 후 Next 버튼을 클릭한다.

>![mysql_vsphere_4.1.04]

<br>

- HeidiSQL을 설치할 경로를 설정 후 Next 버튼을 클릭한다.

>별도의 경로 설정이 필요 없을 경우 default로 C드라이브 Program Files 폴더에 설치가 된다.

>![mysql_vsphere_4.1.05]

<br>

- 설치 완료 후 시작메뉴에 HeidiSQL 바로가기 아이콘의 이름을 설정하는 과정이다.  
>Next 버튼을 클릭하여 다음 과정을 진행한다.

>![mysql_vsphere_4.1.06]

<br>

- 체크박스가 4개가 있다. 아래의 경우를 고려하여 체크 및 해제를 한다.
>
  바탕화면에 바로가기 아이콘을 생성할 경우  
  sql확장자를 HeidiSQL 프로그램으로 실행할 경우  
  heidisql 공식 홈페이지를 통해 자동으로 update check를 할 경우  
  heidisql 공식 홈페이지로 자동으로 버전을 전송할 경우

> 체크박스에 체크 설정/해제를 완료했다면 Next 버튼을 클릭한다.

>![mysql_vsphere_4.1.07]

<br>

- 설치를 위한 모든 설정이 한번에 출력된다. 확인 후 Install 버튼을 클릭하여 설치를 진행한다.

>![mysql_vsphere_4.1.08]

<br>

- Finish 버튼 클릭으로 설치를 완료한다.

>![mysql_vsphere_4.1.09]

<br>

- HeidiSQL을 실행했을 때 처음 뜨는 화면이다. 이 화면에서 Server에 접속하기 위한 profile을 설정/저장하여 접속할 수 있다. 신규 버튼을 클릭한다.

>![mysql_vsphere_4.1.10]

<br>

- 어떤 Server에 접속하기 위한 Connection 정보인지 별칭을 입력한다.

>![mysql_vsphere_4.1.11]

<br>

- 네트워크 유형의 목록에서 MySQL(SSH tunel)을 선택한다.

>![mysql_vsphere_4.1.12]

<br>

- 아래 붉은색 영역에 접속하려는 서버 정보를 모두 입력한다.

>![mysql_vsphere_4.1.13]

>서버 정보는 Application에 바인드되어 있는 서버 정보를 입력한다. cf env <app_name> 명령어로 이용하여 확인한다.

>**예)** $cf env hello-spring-mysql

>![mysql_vsphere_4.1.14]

<br>

- - SSH 터널 탭을 클릭하고 OpenPaaS 운영 관리자에게 제공받은 SSH 터널링 가능한 서버 정보를 입력한다. plink.exe 위치 입력은 Putty에서 제공하는 plink.exe 실행 위치를 넣어주고 만일 해당 파일이 없을 경우 plink.exe 내려받기 링크를 클릭하여 다운받는다. 로컬 포트 정보는 임의로 넣고 열기 버튼을 클릭하면 Mysql 데이터베이스에 접속한다.

>(참고) 만일 개인 키로 접속이 가능한 경우에는 openstack용 Open PaaS Mysql 서비스팩 설치 가이드를 참고한다.

>![mysql_vsphere_4.1.15]

<br>

- 접속이 완료되면 좌측에 스키마 정보가 나타난다. 하지만 초기설정은 테이블, 뷰, 프로시져, 함수, 트리거, 이벤트 등 모두 섞여 있어서 한눈에 구분하기가 힘들어서 접속한 DB 별칭에 마우스 오른쪽 클릭 후 "트리 방식 옵션" - "객체를 유형별로 묶기"를 클릭하면 아래 화면과 같이 각 유형별로 구분이된다.

>![mysql_vsphere_4.1.16]

<br>

- 우측 화면에 쿼리 탭을 클릭하여 Query문을 작성한 후 실행 버튼(삼각형)을 클릭한다.  

>쿼리문에 이상이 없다면 정상적으로 결과를 얻을 수 있을 것이다.

>![mysql_vsphere_4.1.17]
	
	
	
[mysql_vsphere_1.3.01]:./images/mysql/mysql_vsphere_1.3.01.png
[mysql_vsphere_2.2.01]:./images/mysql/mysql_vsphere_2.2.01.png
[mysql_vsphere_2.2.02]:./images/mysql/mysql_vsphere_2.2.02.png
[mysql_vsphere_2.2.03]:./images/mysql/mysql_vsphere_2.2.03.png
[mysql_vsphere_2.2.04]:./images/mysql/mysql_vsphere_2.2.04.png
[mysql_vsphere_2.2.05]:./images/mysql/mysql_vsphere_2.2.05.png
[mysql_vsphere_2.2.06]:./images/mysql/mysql_vsphere_2.2.06.png
[mysql_vsphere_2.2.07]:./images/mysql/mysql_vsphere_2.2.07.png
[mysql_vsphere_2.2.08]:./images/mysql/mysql_vsphere_2.2.08.png
[mysql_vsphere_2.3.01]:./images/mysql/mysql_vsphere_2.3.01.png
[mysql_vsphere_2.3.02]:./images/mysql/mysql_vsphere_2.3.02.png
[mysql_vsphere_2.3.03]:./images/mysql/mysql_vsphere_2.3.03.png
[mysql_vsphere_2.3.04]:./images/mysql/mysql_vsphere_2.3.04.png
[mysql_vsphere_2.3.05]:./images/mysql/mysql_vsphere_2.3.05.png
[mysql_vsphere_2.3.06]:./images/mysql/mysql_vsphere_2.3.06.png
[mysql_vsphere_2.3.07]:./images/mysql/mysql_vsphere_2.3.07.png
[mysql_vsphere_2.4.01]:./images/mysql/mysql_vsphere_2.4.01.png
[mysql_vsphere_2.4.02]:./images/mysql/mysql_vsphere_2.4.02.png
[mysql_vsphere_2.4.03]:./images/mysql/mysql_vsphere_2.4.03.png
[mysql_vsphere_2.4.04]:./images/mysql/mysql_vsphere_2.4.04.png
[mysql_vsphere_2.4.05]:./images/mysql/mysql_vsphere_2.4.05.png
[mysql_vsphere_3.1.01]:./images/mysql/mysql_vsphere_3.1.01.png
[mysql_vsphere_3.2.01]:./images/mysql/mysql_vsphere_3.2.01.png
[mysql_vsphere_3.2.02]:./images/mysql/mysql_vsphere_3.2.02.png
[mysql_vsphere_3.2.03]:./images/mysql/mysql_vsphere_3.2.03.png
[mysql_vsphere_3.3.01]:./images/mysql/mysql_vsphere_3.3.01.png
[mysql_vsphere_3.3.02]:./images/mysql/mysql_vsphere_3.3.02.png
[mysql_vsphere_3.3.03]:./images/mysql/mysql_vsphere_3.3.03.png
[mysql_vsphere_3.3.04]:./images/mysql/mysql_vsphere_3.3.04.png
[mysql_vsphere_3.3.05]:./images/mysql/mysql_vsphere_3.3.05.png
[mysql_vsphere_3.3.06]:./images/mysql/mysql_vsphere_3.3.06.png
[mysql_vsphere_3.3.07]:./images/mysql/mysql_vsphere_3.3.07.png
[mysql_vsphere_3.3.08]:./images/mysql/mysql_vsphere_3.3.08.png
[mysql_vsphere_3.3.09]:./images/mysql/mysql_vsphere_3.3.09.png
[mysql_vsphere_4.1.01]:./images/mysql/mysql_vsphere_4.1.01.png
[mysql_vsphere_4.1.02]:./images/mysql/mysql_vsphere_4.1.02.png
[mysql_vsphere_4.1.03]:./images/mysql/mysql_vsphere_4.1.03.png
[mysql_vsphere_4.1.04]:./images/mysql/mysql_vsphere_4.1.04.png
[mysql_vsphere_4.1.05]:./images/mysql/mysql_vsphere_4.1.05.png
[mysql_vsphere_4.1.06]:./images/mysql/mysql_vsphere_4.1.06.png
[mysql_vsphere_4.1.07]:./images/mysql/mysql_vsphere_4.1.07.png
[mysql_vsphere_4.1.08]:./images/mysql/mysql_vsphere_4.1.08.png
[mysql_vsphere_4.1.09]:./images/mysql/mysql_vsphere_4.1.09.png
[mysql_vsphere_4.1.10]:./images/mysql/mysql_vsphere_4.1.10.png
[mysql_vsphere_4.1.11]:./images/mysql/mysql_vsphere_4.1.11.png
[mysql_vsphere_4.1.12]:./images/mysql/mysql_vsphere_4.1.12.png
[mysql_vsphere_4.1.13]:./images/mysql/mysql_vsphere_4.1.13.png
[mysql_vsphere_4.1.14]:./images/mysql/mysql_vsphere_4.1.14.png
[mysql_vsphere_4.1.15]:./images/mysql/mysql_vsphere_4.1.15.png
[mysql_vsphere_4.1.16]:./images/mysql/mysql_vsphere_4.1.16.png
[mysql_vsphere_4.1.17]:./images/mysql/mysql_vsphere_4.1.17.png



[update_mysql_vsphere_01]:./images/mysql/update_mysql_vsphere_01.png
[update_mysql_vsphere_02]:./images/mysql/update_mysql_vsphere_02.png
[update_mysql_vsphere_03]:./images/mysql/update_mysql_vsphere_03.png
[update_mysql_vsphere_04]:./images/mysql/update_mysql_vsphere_04.png
[update_mysql_vsphere_05]:./images/mysql/update_mysql_vsphere_05.png
[update_mysql_vsphere_06]:./images/mysql/update_mysql_vsphere_06.png
[update_mysql_vsphere_07]:./images/mysql/update_mysql_vsphere_07.png
[update_mysql_vsphere_08]:./images/mysql/update_mysql_vsphere_08.png
[update_mysql_vsphere_09]:./images/mysql/update_mysql_vsphere_09.png
[update_mysql_vsphere_10]:./images/mysql/update_mysql_vsphere_10.png
[update_mysql_vsphere_11]:./images/mysql/update_mysql_vsphere_11.png
[update_mysql_vsphere_12]:./images/mysql/update_mysql_vsphere_12.png
[update_mysql_vsphere_13]:./images/mysql/update_mysql_vsphere_13.png
[update_mysql_vsphere_14]:./images/mysql/update_mysql_vsphere_14.png
[update_mysql_vsphere_15]:./images/mysql/update_mysql_vsphere_15.png

[update_mysql_vsphere_25]:./images/mysql/update_mysql_vsphere_25.png
[update_mysql_vsphere_30]:./images/mysql/update_mysql_vsphere_30.png
[update_mysql_vsphere_31]:./images/mysql/update_mysql_vsphere_31.png
[update_mysql_vsphere_34]:./images/mysql/update_mysql_vsphere_34.png

[update_mysql_vsphere_35]:./images/mysql/update_mysql_vsphere_35.png
[update_mysql_vsphere_36]:./images/mysql/update_mysql_vsphere_36.png
[update_mysql_vsphere_37]:./images/mysql/update_mysql_vsphere_37.png
[update_mysql_vsphere_38]:./images/mysql/update_mysql_vsphere_38.png
[update_mysql_vsphere_39]:./images/mysql/update_mysql_vsphere_39.png
[update_mysql_vsphere_40]:./images/mysql/update_mysql_vsphere_40.png
[update_mysql_vsphere_41]:./images/mysql/update_mysql_vsphere_41.png
[update_mysql_vsphere_42]:./images/mysql/update_mysql_vsphere_42.png
[update_mysql_vsphere_43]:./images/mysql/update_mysql_vsphere_43.png
[update_mysql_vsphere_44]:./images/mysql/update_mysql_vsphere_44.png
[update_mysql_vsphere_45]:./images/mysql/update_mysql_vsphere_45.png
[update_mysql_vsphere_46]:./images/mysql/update_mysql_vsphere_46.png
[update_mysql_vsphere_47]:./images/mysql/update_mysql_vsphere_47.png
[update_mysql_vsphere_48]:./images/mysql/update_mysql_vsphere_48.png
[update_mysql_vsphere_49]:./images/mysql/update_mysql_vsphere_49.png
[update_mysql_vsphere_50]:./images/mysql/update_mysql_vsphere_50.png

### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > MySQL Service
