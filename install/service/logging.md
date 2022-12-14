### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Logging Service

## Table of Contents

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)   
  1.3. [참고자료](#1.3)  

2. [Logging 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)   
  2.2. [Stemcell 확인](#2.2)    
  2.3. [Deployment 다운로드](#2.3)   
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)      
  2.6. [서비스 설치 확인](#2.6)  

3. [Logging 서비스 관리](#3)  
  3.1. [Logging 서비스 활성화](#3.1)  



## <div id="1"/> 1. 문서 개요

### <div id="1.1"/> 1.1. 목적

본 문서(Logging Service 설치 가이드)는 Bosh를 이용하여 Logging Service를 설치 하는 방법을 기술하였다.  
Logging Service를 설치할 경우, **설치된 시점**을 기준으로 cf로 배포한 app의 누적된 log를 확인 할 수 있다.  
보관기간은 운영기관에 따라 상이하며, 기본은 7일이다. 

### <div id="1.2"/> 1.2. 범위

설치 범위는 Logging Service를 검증하기 위한 기본 설치를 기준으로 작성하였다.

### <div id="1.3"/> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

## <div id="2"/> 2. Logging 서비스 설치  

### <div id="2.1"/> 2.1. Prerequisite 

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스 설치를 위해서는 **BOSH**와 **PaaS-TA**, **Portal** 설치가 선행되어야 한다.


### <div id="2.2"/> 2.2. Stemcell 확인  

Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-bionic 1.122를 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells  

```
Using environment '10.0.1.6' as client 'admin'

Name                                     Version  OS             CPI  CID  
bosh-vsphere-esxi-ubuntu-bionic-go_agent  1.122*  ubuntu-bionic  -    sc-c9eeb237-f344-4396-ab29-90bfab2b6a75 

(*) Currently deployed

1 stemcells

Succeeded
```  

만약 해당 Stemcell이 업로드 되어 있지 않다면 [bosh.io 스템셀](https://bosh.io/stemcells/) 에서 해당되는 IaaS환경과 버전에 해당되는 스템셀 링크를 복사 후 다음과 같은 명령어를 실행한다.

```
# Stemcell 업로드 명령어 예제
bosh -e ${BOSH_ENVIRONMENT} upload-stemcell -n {STEMCELL_URL}
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
- `common_vars.yml`을 서버 환경에 맞게 수정한다. 
- Logging 서비스에서 사용하는 변수는 system_domain, uaa_client_admin_id, uaa_client_admin_secret 이다.
> $ vi ~/workspace/common/common_vars.yml
```yaml
... ((생략)) ...

# PAAS-TA INFO
system_domain: "61.252.53.246.nip.io"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
uaa_client_admin_id: "admin"			# UAAC Admin Client Admin ID
uaa_client_admin_secret: "admin-secret"		# UAAC Admin Client에 접근하기 위한 Secret 변수

... ((생략)) ...

```


- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.
> $ vi ~/workspace/service-deployment/logging-service/vars.yml

```yaml
# STEMCELL INFO
stemcell_os: "ubuntu-bionic"		# Stemcell OS
stemcell_version: "1.122"		# Stemcell Version


# VARIABLE
syslog_forwarder_custom_rule: 'if ($msg contains "DEBUG") then stop'      # PaaS-TA Logging Agent에서 전송할 Custom Rule
syslog_forwarder_fallback_servers: []
portal_deploy_type: "vm"                     # PaaS-TA Portal 배포 타입(vm, app)


# Fluentd
fluentd_azs: ["z4"]                    # fluentd : azs
fluentd_instances: 1                   # fluentd : instances (1)
fluentd_vm_type: "small"               # fluentd : vm type
fluentd_network: "default"             # fluentd 네트워크
fluentd_ip: "10.0.1.105"
fluentd_port: "3514"                   # fluentd Port
fluentd_transport: "tcp"               # fluentd Logging Protocol


# INFLUXDB
influxdb_azs: ["z4"]			            # InfluxDB : azs
influxdb_instances: 1			            # InfluxDB : instances (1)
influxdb_vm_type: "large"		          # InfluxDB : vm type
influxdb_network: "default"		        # InfluxDB 네트워크
influxdb_persistent_disk_type: "10GB"	# InfluxDB 영구 Disk 종류

influxdb_ip: "10.0.1.115"
influxdb_http_port: "8086"                  # default 8086
influxdb_username: "admin"	  # InfluxDB Admin 계정 Username
influxdb_password: "PaaS-TA2022"	  # InfluxDB Admin 계정 Password
influxdb_interval: "7d"                     # InfluxDB Retention Policy (bootstrapper)
influxdb_https_enabled: "true"              # InfluxDB HTTPS 설정

influxdb_database: "logging_db"          # InfluxDB Database명
influxdb_measurement: "logging_measurement"    # InfluxDB Measurement명
influxdb_time_precision: "s"    # hour(h), minutes(m), second(s), millisecond(ms), microsecond(u), nanosecond(ns)
influxdb_query_limit: "50"                  # InfluxDB query limit (default "50")


# COLLECTOR
collector_azs: ["z4"]           # collector : azs
collector_instances: 1          # collector : instances (1)
collector_vm_type: "small"      # collector : vm type
collector_network: "default"    # collector 네트워크


# LOG_API
log_api_azs: ["z4"]                                             # log-api : azs
log_api_instances: 1                                            # log-api : instances (1)
log_api_vm_type: "small"                                        # log-api : vm type
log_api_network: "default"                                      # log-api 네트워크
```  

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정한다.
> $ vi ~/workspace/portal-deployment/logging-service/deploy.sh

```shell script
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"             # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)


# Portal 설치 타입 및 프로토콜 종류에 따라 옵션 파일 사용 여부를 분기한다.
PORTAL_TYPE=`grep portal_deploy_type vars.yml | cut -d "#" -f1`
FLUENTD_TRANSPORT=`grep fluentd_transport vars.yml`


if [[ "${PORTAL_TYPE}" =~ "app" ]]; then
  if [[ "${FLUENTD_TRANSPORT}" =~ "tcp" ]]; then
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -o operations/portal-app-type.yml \
          -o operations/use-protocol-tcp.yml \
          -l vars.yml \
          -l ${COMMON_VARS_PATH}
  else
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -o operations/portal-app-type.yml \
          -l vars.yml \
          -l ${COMMON_VARS_PATH}
  fi
elif [[ "${PORTAL_TYPE}" =~ "vm" ]]; then
  if [[ "${FLUENTD_TRANSPORT}" =~ "tcp" ]]; then
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -o operations/use-protocol-tcp.yml \
          -l vars.yml \
          -l ${COMMON_VARS_PATH}
  else
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -l vars.yml \
          -l ${COMMON_VARS_PATH}
  fi
else
  echo "Logging Service can't install. Please check 'portal_deploy_type'."
fi
```

- 서비스를 설치한다.  
```shell
$ cd ~/workspace/service-deployment/logging-service
$ source deploy.sh  
```  


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} -d logging-service vms
```shell
Using environment '10.0.1.6' as client 'admin'

Task 193. Done

Deployment 'logging-service'

Instance                                        Process State  AZ  IPs         VM CID                                   VM Type  Active  Stemcell  
log-api/0d710aae-3c0f-44b2-ac79-4a134ad2c601    running        z4  10.0.1.163  vm-d04c9c5f-55c8-44ca-b64b-57db7c60d619  small    true    bosh-vsphere-esxi-ubuntu-bionic-go_agent/1.122  
collector/c417b48c-4efe-47fd-a9a0-70529b6963cc  running        z4  10.0.1.162  vm-16a6a6b0-db47-4feb-b409-6bbd27370774  small    true    bosh-vsphere-esxi-ubuntu-bionic-go_agent/1.122  
fluentd/0e7912d8-e9b0-4bab-986b-8f2c9351af77    running        z4  10.0.1.105  vm-758fdead-9b51-4fcf-aea9-929954dddee4  small    true    bosh-vsphere-esxi-ubuntu-bionic-go_agent/1.122  
influxdb/e4078a4d-b71b-4e91-8f30-85189c3ecf3f   running        z4  10.0.1.115  vm-7d3b4e7f-eb90-406e-b2a4-c16c458d4bdc  large    true    bosh-vsphere-esxi-ubuntu-bionic-go_agent/1.122

```

## <div id="3"/>3.  Logging 서비스 관리

서비스 설치가 완료 되면, PaaS-TA 포탈에서 서비스를 사용하기 위해 Logging 서비스 활성화 코드 등록을 해 주어야 한다.

## <div id="3.1"/>  3.1. Logging 서비스 활성화

-	PaaS-TA 운영자 포탈에 접속한다.
![002]

-	운영관리의 코드관리 메뉴로 이동하여 다음과 같이 코드를 등록한다.

> ※ Group Table  
> 코드 ID  : LOGGING  
> 코드 이름 : Logging Service  
> ![003]
>
> ※ Detail Table  
> Key : enable_logging_service  
> Value : true  
> 요약 : Logging Service Enable Code  
> 사용 : Y  
> ![004]

![005]

[001]:./images/logging-service/image001.png
[002]:./images/logging-service/image002.png
[003]:./images/logging-service/image003.png
[004]:./images/logging-service/image004.png
[005]:./images/logging-service/image005.png

### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Logging Service
