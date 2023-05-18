### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Portal Container Type

## Table of Contents  

1. [문서 개요](#1)  
    1.1. [목적](#1.1)  
    1.2. [범위](#1.2)  
    1.3. [참고자료](#1.3)  

2. [PaaS-TA AP Portal infra 설치](#2)  
    2.1. [Prerequisite](#2.1)   
    2.2. [Stemcell 확인](#2.2)    
    2.3. [Deployment 다운로드](#2.3)   
    2.4. [Deployment 파일 수정](#2.4)  
    2.5. [서비스 설치](#2.5)    
    2.6. [서비스 설치 확인](#2.6)  

3. [PaaS-TA AP Portal 설치](#3)  
    3.1. [Portal App 구성](#3.1)  
    3.2. [Portal App 배포 Script 변수 설정](#3.2)  
    3.3. [Portal App 배포 Script 실행](#3.3)  

4. [PaaS-TA AP Portal 운영](#4)  
    4.1. [사용자의 조직 생성 Flag 활성화](#4.1)  
    4.2. [사용자포탈 UAA 페이지 오류](#4.2)  
    4.3. [카탈로그 적용](#4.3)  


## <div id="1"/> 1. 문서 개요
### <div id="1.1"/> 1.1. 목적

본 문서(PaaS-TA AP Portal Container Type 설치 가이드)는 PaaS-TA AP Portal을 BOSH와 PaaS-TA AP를 이용하여 설치 하는 방법을 기술하였다.

### <div id="1.2"/> 1.2. 범위
설치 범위는 PaaS-TA AP Portal을 검증하기 위한 Portal infra 설치 및 Portal App 배포를 기준으로 작성하였다.

### <div id="1.3"/> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

## <div id="2"/> 2. PaaS-TA AP Portal infra 설치  

### <div id="2.1"/> 2.1. Prerequisite
본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.  
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.  
UAA client가 설치 되어 있지 않을 경우 UAA client의 설치가 필요하다.

- UAA client 설치 (BOSH Dependency 설치 필요)
```
$ sudo gem install cf-uaac
$ uaac -v
```

### <div id="2.2"/> 2.2. Stemcell 확인
Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-bionic 1.195를 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version   OS             CPI  CID  
bosh-openstack-kvm-ubuntu-bionic-go_agent  1.195      ubuntu-bionic  -    ce507ae4-aca6-4a6d-b7c7-220e3f4aaa7d

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

- Portal Deployment Git Repository URL : https://github.com/PaaS-TA/portal-deployment/tree/v5.2.21

```
# Deployment 다운로드 파일 위치 경로 생성 및 설치 경로 이동
$ mkdir -p ~/workspace
$ cd ~/workspace

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/portal-deployment.git -b v5.2.21
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

- common_vars.yml을 서버 환경에 맞게 수정한다.
- PaaS-TA AP Portal infra에서 사용하는 변수는 system_domain, portal_web_user_language, portal_web_admin_language 이다.

> $ vi ~/workspace/common/common_vars.yml
```
... ((생략)) ...

system_domain: "61.252.53.246.nip.io"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
portal_web_user_language: ["ko", "en"]             # portal webuser language list (e.g. ["ko", "en"])
portal_web_admin_language: ["ko", "en"]             # portal webadmin language list (e.g. ["ko", "en"])

... ((생략)) ...
```


- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/portal-deployment/portal-container-infra/vars.yml

```
# STEMCELL INFO
stemcell_os: "ubuntu-bionic"                                    # stemcell os
stemcell_version: "1.195"                                        # stemcell version

# NETWORKS INFO
private_networks_name: "default"                                # private network name

# PORTAL-INFRA INFO
infra_azs: [z3]                                                 # infra : azs
infra_instances: 1                                              # infra : instances (1)
infra_vm_type: "large"                                          # infra : vm type
infra_persistent_disk_type: "20GB"                              # infra : persistent disk type


# MARIADB INFO
mariadb_port: "<MARIADB_PORT>"                                  # mariadb : database port (e.g. 13306) -- Do Not Use "3306"
mariadb_admin_password: "<MARIADB_ADMIN_PASSWORD>"              # mariadb : database admin password (e.g. "Paasta@2019")
portal_default_api_name: "PaaS-TA"                              # portal default api name (e.g. PaaS-TA {Version})
portal_default_api_url: "http://<PORTAL_GATEWAY_ROUTE>"         # portal default api url (portal gateway url) (e.g. "http://portal-gateway.<DOMAIN>")
portal_default_header_auth: "Basic YWRtaW46b3BlbnBhYXN0YQ=="    # portal default header auth
portal_default_api_desc: "PaaS-TA infra"                        # portal default api description (e.g. PaaS-TA {Version} infra)


# BINARY_STORAGE INFO
binary_storage_auth_port: "<BINARY_STORAGE_AUTH_PORT>"          # binary storage : keystone port (e.g. 15001) -- Do Not Use "5000"
binary_storage_username: "<BINARY_STORAGE_USERNAME>"            # binary storage : username (e.g. "paasta-portal")
binary_storage_password: "<BINARY_STORAGE_PASSWORD>"            # binary storage : password (e.g. "paasta")
binary_storage_tenantname: "<BINARY_STORAGE_TENANTNAME>"        # binary storage : tenantname (e.g. "paasta-portal")
binary_storage_email: "<BINARY_STORAGE_EMAIL>"                  # binary storage : email (e.g. "paasta@paasta.com")
```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/portal-deployment/portal-container-infra/deploy.sh
```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"  # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"      # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d portal-container-infra deploy --no-redact portal-container-infra.yml \
   -o operations/cce.yml \
   -l ${COMMON_VARS_PATH} \
   -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/portal-deployment/portal-container-infra    
$ sh ./deploy.sh  
```


### <div id="2.6"/> 2.6. 서비스 설치 확인
설치 완료된 서비스를 확인한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} -d portal-container-infra vms  

```
Using environment '10.0.1.6' as client 'admin'

Task 1246. Done

Deployment 'portal-container-infra'

Instance                                    Process State  AZ  IPs          VM CID               VM Type  Active  
infra/3193a1fa-156d-4dd9-935d-4b67cdcc1182  running        z3  10.0.81.121  i-09ec0c2e7f3594683  large    true  

1 vms

Succeeded
```

## <div id="3"/> 3. PaaS-TA AP Portal 설치
### <div id="3.1"/> 3.1. Portal App 구성
PaaS-TA AP에 Portal 관련 App이 9개 배포되며 구성은 다음과 같다.
```
portal-app-1.2.9
├── portal-api-2.4.2
├── portal-common-api-2.2.5
├── portal-gateway-2.2.1
├── portal-log-api-2.3.0
├── portal-registration-2.1.0
├── portal-ssh-1.0.0
├── portal-storage-api-2.2.1
├── portal-web-admin-2.3.5
└── portal-web-user-2.4.6
```
### <div id="3.2"/> 3.2. Portal App 배포 Script 변수 설정  
Portal App 배포 Script 실행을 위하여 Script가 있는 위치로 이동한다.

```
### 설치 작업 경로 이동
$ cd ~/workspace/portal-deployment/portal-container-infra/scripts
```

Script 변수를 설정한다. (FILE PATH 부분 변경 필수, 나머지 옵션)
> $ vi portal-app-variable.yml
```
#!/bin/bash

##FILE PATH
COMMON_VARS_PATH=~/workspace/common/common_vars.yml	# common_vars.yml Path
PORTAL_INFRA_VARS_PATH=~/workspace/portal-deployment/portal-container-infra/vars.yml	# portal_infra_vars.yml Path
PORTAL_APP_WORKING_DIRECTORY=~/workspace/portal-deployment/portal-container-infra/portal-app # Portal APP Working Path


##PORTAL VARIABLE
USER_APP_SIZE_MB=0					# USER My App size(MB), if value==0 -> unlimited
MONITORING_ENABLE=false					# Monitoring Enable Option

PORTAL_ORG_NAME="portal"				# PaaS-TA Portal Org Name
PORTAL_SPACE_NAME="system"				# PaaS-TA Portal Space Name
PORTAL_QUOTA_NAME="portal_quota"			# PaaS-TA Portal Quota Name
PORTAL_SECURITY_GROUP_NAME="portal"			# PaaS-TA Portal Security Group Name

MAIL_SMTP_HOST="smtp.gmail.com"				# Mail-SMTP Host
MAIL_SMTP_PORT="465"					# Mail-SMTP Port
MAIL_SMTP_USERNAME="paasta"				# Mail-SMTP User Name
MAIL_SMTP_PASSWORD="paasta"				# Mail-SMTP Password
MAIL_SMTP_USEREMAIL="paas-ta@gmail.com"			# Mail-SMTP User Email

##PORTAL APP INSTANCES
PORTAL_API_INSTANCE=1					# PORTAL-API INSTANCES
PORTAL_COMMON_API_INSTANCE=1				# PORTAL-COMMON-API INSTANCES
PORTAL_GATEWAY_INSTANCE=1				# PORTAL-GATEWAY INSTANCES
PORTAL_REGISTRATION_INSTANCE=1				# PORTAL-REGISTRATION INSTANCES
PORTAL_STORAGE_API_INSTANCE=1				# PORTAL-STORAGE-API INSTANCES
PORTAL_WEB_ADMIN_INSTANCE=1				# PORTAL-WEB-ADMIN INSTANCES
PORTAL_WEB_USER_INSTANCE=1				# PORTAL-WEB-USER INSTANCES


##UNCHANGE VARIABLE(if defulat install, don't change variable)
PAASTA_DEPLOYMENT_TYPE="ap"       # PaaS TA Deployment Type
PAASTA_CORE_DEPLOYMENT_NAME="paasta"			# PaaS TA AP Deployment Name
PORTAL_INFRA_DEPLOYMENT_NAME="portal-container-infra"	# Portal Container Infra Deployment Name
PAASTA_DATABASE_INSTANCE_NAME="database"		# PaaS TA AP Database Instance Name
PORTAL_INFRA_DATABASE_NAME="infra"			# Portal Container Infra Database Name
UAA_ADMIN_CLIENT_ID="admin"				# UAA Client ID
CC_DB_NAME="cloud_controller"				# PaaS-TA AP CCDB Name
CC_DB_USER_NAME="cloud_controller"			# PaaS-TA AP CCDB ID
UAA_DB_NAME="uaa"					# PaaS-TA AP UAADB Name
UAA_DB_USER_NAME="uaa"					# PaaS-TA AP UAADB ID
UAAC_PORTAL_CLIENT_ID="portalclient"			# UAAC Portal Client ID

IS_PAAS_TA_EXTERNAL_DB=false				# (true or false)
PAAS_TA_EXTERNAL_DB_IP=					# PaaS-TA AP External DB IP
PAAS_TA_EXTERNAL_DB_PORT=				# PaaS-TA AP External DB Port
PAAS_TA_EXTERNAL_DB_KIND=				# PaaS-TA AP External DB Kind(IF USE e.g. postgres or mysql)
IS_PORTAL_EXTERNAL_DB=false				# (true or false)
PORTAL_EXTERNAL_DB_IP=					# Portal External DB IP
PORTAL_EXTERNAL_DB_PORT=				# Portal External DB Port
PORTAL_EXTERNAL_DB_PASSWORD=				# Portal External DB Password
IS_PORTAL_EXTERNAL_STORAGE=false			# (true or false)
PORTAL_EXTERNAL_STORAGE_IP=				# Portal External Storage IP
PORTAL_EXTERNAL_STORAGE_PORT=				# Portal External Storage Port
PORTAL_EXTERNAL_STORAGE_TENANTNAME=			# Portal External Storage Tenant Name
PORTAL_EXTERNAL_STORAGE_USERNAME=			# Portal External Storage Username
PORTAL_EXTERNAL_STORAGE_PASSWORD=			# Portal External Storage Password
```


### <div id="3.3"/> 3.3. Portal App 배포 Script 실행
변수 설정이 완료되었으면 배포 Script를 실행한다.
> $ source deploy-portal-app.sh
```
.....
.....

name                  requested state   processes           routes
portal-api            started           web:1/1, task:0/0   portal-api.61.252.53.246.nip.io
portal-common-api     started           web:1/1, task:0/0   portal-common-api.61.252.53.246.nip.io
portal-gateway        started           web:1/1, task:0/0   portal-gateway.61.252.53.246.nip.io
portal-log-api        started           web:1/1, task:0/0   portal-log-api.61.252.53.246.nip.io
portal-registration   started           web:1/1, task:0/0   portal-registration.61.252.53.246.nip.io
portal-storage-api    started           web:1/1, task:0/0   portal-storage-api.61.252.53.246.nip.io
portal-web-admin      started           web:1/1, task:0/0   portal-web-admin.61.252.53.246.nip.io
portal-web-user       started           web:1/1             portal-web-user.61.252.53.246.nip.io
ssh-app               started           web:1/1             ssh-app.61.252.53.246.nip.io
```



## <div id="4"/>4. PaaS-TA AP Portal 운영

### <div id="4.1"/> 4.1. 사용자의 조직 생성 Flag 활성화

PaaS-TA는 기본적으로 일반 사용자는 조직을 생성할 수 없도록 설정되어 있다. 포털 배포를 위해 조직 및 공간을 생성해야 하고 또 테스트를 구동하기 위해서도 필요하므로 사용자가 조직을 생성할 수 있도록 user_org_creation FLAG를 활성화 한다. FLAG 활성화를 위해서는 PaaS-TA 운영자 계정으로 로그인이 필요하다.

```
$ cf enable-feature-flag user_org_creation
```
```
Setting status of user_org_creation as admin...
OK

Feature user_org_creation Enabled.
```

### <div id="4.2"/> 4.2. 사용자포탈 UAA페이지 오류  

- uaac의 endpoint를 설정하고 uaac 로그인을 실행한다.
```
# endpoint 설정
$ uaac target https://uaa.<DOMAIN> --skip-ssl-validation

# target 확인
$ uaac target
Target: https://uaa.<DOMAIN>
Context: uaa_admin, from client uaa_admin

# uaac 로그인
$ uaac token client get <UAA_CLIENT_ADMIN_ID> -s <UAA_CLIENT_ADMIN_SECRET>
Successfully fetched token via client credentials grant.
Target: https://uaa.<DOMAIN>
Context: admin, from client admin
```
- redirect오류 - portalclient 미등록  
![paas-ta-portal-31]  
1. uaac portalclient가 등록이 되어있지 않다면 해당 화면과 같이 redirect오류가 발생한다.  
2. uaac client add를 통해 potalclient를 추가시켜주어야 한다.   
> $ uaac client add <PORTAL_UAA_CLIENT_ID> -s <PORTAL_UAA_CLIENT_SECRET> --redirect_uri <PORTAL_WEB_USER_URI>, <PORTAL_WEB_USER_URI>/callback --scope   "cloud_controller_service_permissions.read , openid , cloud_controller.read , cloud_controller.write , cloud_controller.admin" --authorized_grant_types "authorization_code , client_credentials , refresh_token" --authorities="uaa.resource" --autoapprove="openid , cloud_controller_service_permissions.read"  

```
# e.g. portal client 계정 생성

$ uaac client add portalclient -s clientsecret --redirect_uri "http://portal-web-user.<DOMAIN>, http://portal-web-user.<DOMAIN>/callback" \
--scope "cloud_controller_service_permissions.read , openid , cloud_controller.read , cloud_controller.write , cloud_controller.admin" \
--authorized_grant_types "authorization_code , client_credentials , refresh_token" \
--authorities="uaa.resource" \
--autoapprove="openid , cloud_controller_service_permissions.read"
```

- redirect오류 - portalclient의 redirect_uri 등록 오류  
![paas-ta-portal-32]  
1. uaac portalclient가 uri가 잘못 등록되어있다면 해당 화면과 같이 redirect오류가 발생한다.   
2. uaac client update를 통해 uri를 수정해야한다.  
> $ uaac client update portalclient --redirect_uri "<PORTAL_WEB_USER_URI>, <PORTAL_WEB_USER_URI>/callback"   

```
#  e.g. portal client redirect_uri update
$ uaac client update portalclient --redirect_uri "http://portal-web-user.<DOMAIN>, http://portal-web-user.<DOMAIN>/callback"
```

### <div id="4.3"/> 4.3. 카탈로그 적용  
##### 1. Catalog 빌드팩, 서비스팩 추가  
Paas-TA Portal 설치 후에 관리자 포탈에서 빌드팩, 서비스팩을 등록해야 사용자 포탈에서 사용이 가능하다.  
- [카탈로그 이미지 다운로드](https://nextcloud.paas-ta.org/index.php/s/EmzfJw38H4GQKTr/download)

1. 관리자 포탈에 접속한다.(portal-web-admin.\<DOMAIN\>)  
![paas-ta-portal-15]  
2. 운영관리를 누른다.  
![paas-ta-portal-16]  
3. 카탈로그 페이지에 들어간다.  
![paas-ta-portal-17]  
4. 빌드팩, 서비스팩 상세화면에 들어가서 각 항목란에 값을 입력후에 저장을 누른다.  
![paas-ta-portal-18]  
※ 카탈로그 등록 및 수정 시 카탈로그 관리 코드는 선택 필수이며, 현재 사용 가능한 코드가 없는 경우 다음 내용을 참고하여 처리하도록 한다.
    1. ①"코드 관리"를 클릭한다.
    2. **Group Table**에서 해당하는 ②"분류 코드"를 클릭한다.
    3. **Detail Table**에 ③"등록"버튼을 클릭하여 카탈로그 관리 코드를 추가 후 사용한다.
    ![paas-ta-portal-18-1]
5. 사용자포탈에서 변경된값이 적용되어있는지 확인한다.  
![paas-ta-portal-19]   

[paas-ta-portal-01]:./images/Paas-TA-Portal_App_01.png
[paas-ta-portal-15]:./images/Paas-TA-Portal_15.png
[paas-ta-portal-16]:./images/Paas-TA-Portal_16.png
[paas-ta-portal-17]:./images/Paas-TA-Portal_17.png
[paas-ta-portal-18]:./images/Paas-TA-Portal_18.png
[paas-ta-portal-18-1]:./images/Paas-TA-Portal_18-1.png
[paas-ta-portal-19]:./images/Paas-TA-Portal_19.png
[paas-ta-portal-31]:./images/Paas-TA-Portal_27.jpg
[paas-ta-portal-32]:./images/Paas-TA-Portal_28.jpg

### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Portal Container Type
