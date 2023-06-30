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
  2.7. [Portal Log API 배포](#2.7)  
  2.7.1 [Portal VM Type](#2.7.1)  
  2.7.2 [Portal App Type](#2.7.2)  

3. [Logging 서비스 관리](#3)  
  3.1. [Logging 서비스 활성화](#3.1)  


<br>

# <div id="1"/> 1. 문서 개요

## <div id="1.1"/> 1.1. 목적

본 문서(Logging Service 설치 가이드)는 Bosh를 이용하여 Logging Service를 설치 하는 방법을 기술하였다.  
Logging Service를 설치할 경우, **설치된 시점**을 기준으로 cf로 배포한 app의 누적된 log를 확인 할 수 있다.  
보관기간은 운영기관에 따라 상이하며, 기본은 7일이다. 

## <div id="1.2"/> 1.2. 범위

설치 범위는 Logging Service를 검증하기 위한 기본 설치를 기준으로 작성하였다.

## <div id="1.3"/> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

<br>

# <div id="2"/> 2. Logging 서비스 설치  

## <div id="2.1"/> 2.1. Prerequisite 

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스 설치를 위해서는 **BOSH**와 **PaaS-TA**, **Portal** 설치가 선행되어야 한다.


## <div id="2.2"/> 2.2. Stemcell 확인  

Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-jammy 1.102를 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells  

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version  OS             CPI  CID  
bosh-openstack-kvm-ubuntu-jammy-go_agent   1.102*   ubuntu-jammy   -    5f8c7470-e948-4a6f-99e6-5dffce49026f

(*) Currently deployed

1 stemcells

Succeeded
```  

만약 해당 Stemcell이 업로드 되어 있지 않다면 [bosh.io 스템셀](https://bosh.io/stemcells/) 에서 해당되는 IaaS환경과 버전에 해당되는 스템셀 링크를 복사 후 다음과 같은 명령어를 실행한다.

```
# Stemcell 업로드 명령어 예제
bosh -e ${BOSH_ENVIRONMENT} upload-stemcell -n {STEMCELL_URL}
```

## <div id="2.3"/> 2.3. Deployment 다운로드

서비스 설치에 필요한 Deployment를 Git Repository에서 받아 서비스 설치 작업 경로로 위치시킨다.  

- Service Deployment Git Repository URL : https://github.com/PaaS-TA/service-deployment/tree/v5.1.22

```
# Deployment 다운로드 파일 위치 경로 생성 및 설치 경로 이동
$ mkdir -p ~/workspace
$ cd ~/workspace

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/service-deployment.git -b v5.1.22
```

## <div id="2.4"/> 2.4. Deployment 파일 수정
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
stemcell_os: "ubuntu-jammy"                     # Stemcell OS
stemcell_version: "1.102"                       # Stemcell Version


# VARIABLE
syslog_forwarder_custom_rule: 'if ($msg contains "DEBUG") then stop'    # PaaS-TA Logging Agent에서 전송할 Custom Rule
syslog_forwarder_fallback_servers: []
portal_deploy_type: "vm"                        # PaaS-TA Portal 배포 타입(vm, app)


# Fluentd
fluentd_azs: ["z4"]                             # fluentd : azs
fluentd_instances: 1                            # fluentd : instances (1)
fluentd_vm_type: "small"                        # fluentd : vm type
fluentd_network: "default"                      # fluentd 네트워크
fluentd_ip: "10.0.1.105"
fluentd_port: "3514"                            # fluentd Port
fluentd_transport: "tcp"                        # fluentd Logging Protocol


# INFLUXDB
influxdb_azs: ["z4"]                            # InfluxDB : azs
influxdb_instances: 1                           # InfluxDB : instances (1)
influxdb_vm_type: "large"                       # InfluxDB : vm type
influxdb_network: "default"                     # InfluxDB 네트워크
influxdb_persistent_disk_type: "10GB"           # InfluxDB 영구 Disk 종류

influxdb_ip: "10.0.1.115"
influxdb_http_port: "8086"                      # default 8086
influxdb_username: "admin"                      # InfluxDB Admin 계정 Username
influxdb_password: "PaaS-TA2022"                # InfluxDB Admin 계정 Password
influxdb_interval: "7d"                         # InfluxDB Retention Policy (bootstrapper)
influxdb_https_enabled: "true"                  # InfluxDB HTTPS 설정

influxdb_database: "logging_db"                 # InfluxDB Database명
influxdb_measurement: "logging_measurement"     # InfluxDB Measurement명
influxdb_time_precision: "s"                    # hour(h), minutes(m), second(s), millisecond(ms), microsecond(u), nanosecond(ns)


# COLLECTOR
collector_azs: ["z4"]                           # collector : azs
collector_instances: 1                          # collector : instances (1)
collector_vm_type: "small"                      # collector : vm type
collector_network: "default"                    # collector 네트워크
```  

## <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정한다.
> $ vi ~/workspace/portal-deployment/logging-service/deploy.sh

```shell script
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"              # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                  # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)


# Portal 설치 타입 및 프로토콜 종류에 따라 옵션 파일 사용 여부를 분기한다.
PORTAL_DEPLOY_TYPE=`grep portal_deploy_type vars.yml | cut -d "#" -f1`
FLUENTD_TRANSPORT=`grep fluentd_transport vars.yml`


if [[ "${PORTAL_DEPLOY_TYPE}" =~ "app" ]]; then
  if [[ "${FLUENTD_TRANSPORT}" =~ "tcp" ]]; then
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -o operations/use-protocol-tcp.yml \
          -l vars.yml \
          -l operations/pem.yml \
          -l ${COMMON_VARS_PATH}
  else
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -l vars.yml \
          -l operations/pem.yml \
          -l ${COMMON_VARS_PATH}
  fi
elif [[ "${PORTAL_DEPLOY_TYPE}" =~ "vm" ]]; then
  if [[ "${FLUENTD_TRANSPORT}" =~ "tcp" ]]; then
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -o operations/use-protocol-tcp.yml \
          -l vars.yml \
          -l operations/pem.yml \
          -l ${COMMON_VARS_PATH}
  else
    bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
          -l vars.yml \
          -l operations/pem.yml \
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


## <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} -d logging-service vms
```shell
Using environment '10.0.1.6' as client 'admin'

Task 193. Done

Deployment 'logging-service'

Instance                                        Process State  AZ  IPs         VM CID                                VM Type  Active  Stemcell  
collector/542d5b3f-6243-4bea-a8a9-185edba2cbee  running        z1  10.0.1.162  67ad8e85-2ac2-420e-bb71-14fbd7e43013  small    true    bosh-openstack-kvm-ubuntu-jammy-go_agent/1.102  
fluentd/4ec53005-6390-401f-a569-285d1d9bcb95    running        z1  10.0.1.105  2f5d3ab2-787e-4034-ac6f-fb5b75fc29ec  small    true    bosh-openstack-kvm-ubuntu-jammy-go_agent/1.102  
influxdb/e8fc41b5-e5ed-4176-ad5b-216d2cccf88b   running        z1  10.0.1.115  7654584c-ee0c-42df-bef7-d070362eb878  large    true    bosh-openstack-kvm-ubuntu-jammy-go_agent/1.102

```


## <div id="2.7"/> 2.7. Portal Log API 배포

Logging 서비스를 활성화 하기 위해서는 Portal Log API를 배포해야 한다.  
Portal 배포 시 Log API를 배포했다면 [3. Logging 서비스 관리](#3)부터 진행한다.

### <div id="2.7.1"/> 2.7.1. Portal VM Type

- Portal API에서 사용하는 변수 파일을 수정한다. 

> $ vi ~/workspace/portal-deployment/portal-api/vars.yml

```yaml
...

# PORTAL_LOG_API INFO
log_api_azs: [z6]                                               # portal-log-api : azs
log_api_instances: 1                                            # portal-log-api : instances (1)
log_api_vm_type: "small"                                        # portal-log-api : vm type
log_api_infra_admin: false                                      # portal-log-api : infra admin (default "false")

log_api_influxdb_ip: "10.0.1.115"                               # portal-log-api : InfluxDB IP
log_api_influxdb_http_port: "8086"                              # portal-log-api : InfluxDB HTTP PORT (default 8086)
log_api_influxdb_username: "admin"                              # portal-log-api : InfluxDB Admin 계정 Username
log_api_influxdb_password: "PaaS-TA2022"                        # portal-log-api : InfluxDB Admin 계정 Password
log_api_influxdb_https_enabled: true                            # portal-log-api : InfluxDB HTTPS 설정 (default "true")

log_api_influxdb_database: "logging_db"                         # portal-log-api : InfluxDB Database명
log_api_influxdb_measurement: "logging_measurement"             # portal-log-api : InfluxDB Measurement명
log_api_influxdb_query_limit: "50"                              # portal-log-api : InfluxDB query limit (default "50")

...
```

- 서비스를 설치한다.  

```
$ cd ~/workspace/portal-deployment/portal-api   
$ sh ./deploy.sh  
```

- 설치 완료된 서비스를 확인한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} -d portal-api vms  

```diff
  Using environment '10.0.1.6' as client 'admin'

  Task 4823. Done

  Deployment 'portal-api'

  Instance                                                          Process State  AZ  IPs            VM CID                                   VM Type        Active  
  binary_storage/9f58a9b7-2a3d-4ee9-8975-7b04b99c0a21               running        z5  10.30.107.212  vm-e65ad396-ce65-4ef0-962d-5c54fa411769  portal_large   true  
  haproxy/8cc2d633-2b43-4f3d-a2e8-72f5279c11d5                      running        z5  10.30.107.213  vm-315bfa1b-9829-46de-a19d-3bd65e9f9ad4  portal_large   true  
                                                                                       115.68.46.214
  mariadb/117cbf05-b223-4133-bf61-e15f16494e21                      running        z5  10.30.107.211  vm-bc5ae334-12d4-41d4-8411-d9315a96a305  portal_large   true  
  paas-ta-portal-api/48fa0c5a-52eb-4ae8-a7b9-91275615318c           running        z5  10.30.107.217  vm-9d2a1929-0157-4c77-af5e-707ec496ed87  portal_medium  true  
  paas-ta-portal-common-api/060320fa-7f26-4032-a1d9-6a7a41a044a8    running        z5  10.30.107.219  vm-f35e9838-74cf-40e0-9f97-894b53a68d1f  portal_medium  true  
  paas-ta-portal-gateway/6baba810-9a4a-479d-98b2-97e5ba651784       running        z5  10.30.107.214  vm-7ec75160-bf34-442e-b755-778ae7dd3fec  portal_medium  true  
+ paas-ta-portal-log-api/a4460008-42b5-4ba0-84ee-fff49fe6c1bd       running        z5  10.30.107.218  vm-9ec0a1b0-09f6-415b-8e23-53af91fd94b8  portal_medium  true  
  paas-ta-portal-registration/3728ed73-451e-4b93-ab9b-c610826c3135  running        z5  10.30.107.215  vm-c4020514-c458-41c6-bcbc-7e0ee1bc6f42  portal_small   true  
  paas-ta-portal-storage-api/2940366a-8294-4509-a9c0-811c8140663a   running        z5  10.30.107.220  vm-79ad6ee1-1bb5-4308-8b71-9ed30418e2c1  portal_medium  true  
  ...

  9 vms

  Succeeded
```


### <div id="2.7.2"/> 2.7.2. Portal App Type

- Portal App에서 사용하는 Manifest 파일을 수정한다.

> $ vi ~/workspace/portal-container-infra/portal-app/portal-app-1.2.13/portal-log-api-2.3.2/manifest.yml

```yaml
applications:
- name: portal-log-api
  memory: 1G
  instances: 1
  buildpacks:
  - java_buildpack
  path: paas-ta-portal-log-api.jar
  env:

    ...

    ### logging info (InfluxDB)
    influxdb_ip: 10.0.1.115
    influxdb_url: https://10.0.1.42:9096
    influxdb_username: joy
    influxdb_password: joy
    influxdb_database: logging_https
    influxdb_measurement: logging_app_udp
    influxdb_limit: 30
    influxdb_httpsEnabled: true
```

- App을 배포한다.  

```
$ cd ~/workspace/portal-container-infra/portal-app/portal-app-1.2.13/portal-log-api-2.3.2   
$ cf push  

Pushing app portal-log-api to org portal / space system as admin...
Applying manifest file /home/ubuntu/workspace/portal-deployment/portal-container-infra/portal-app/portal-app-1.2.13/portal-log-api-2.3.2/manifest.yml...

...

Waiting for app portal-log-api to start...

Instances starting...
Instances starting...
Instances starting...
Instances starting...
Instances starting...
Instances starting...
Instances starting...

name:              portal-log-api
requested state:   started
routes:            portal-log-api.61.252.53.246.nip.io
last uploaded:     Wed 28 Jun 04:01:52 UTC 2023
stack:             cflinuxfs3
buildpacks:        
	name             version                                                         detect output   buildpack name
	java_buildpack   v4.50-git@github.com:cloudfoundry/java-buildpack.git#5fe41f89   java            java

type:            web
sidecars:        
instances:       1/1
memory usage:    1024M
start command:   JAVA_OPTS="-agentpath:$PWD/.java-buildpack/open_jdk_jre/bin/jvmkill-1.17.0_RELEASE=printHeapHistogram=1 -Djava.io.tmpdir=$TMPDIR -XX:ActiveProcessorCount=$(nproc)
                 -Djava.ext.dirs=$PWD/.java-buildpack/container_security_provider:$PWD/.java-buildpack/open_jdk_jre/lib/ext -Djava.security.properties=$PWD/.java-buildpack/java_security/java.security $JAVA_OPTS" &&
                 CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-3.13.0_RELEASE -totMemory=$MEMORY_LIMIT -loadedClasses=22025 -poolType=metaspace -stackThreads=250
                 -vmOptions="$JAVA_OPTS") && echo JVM Memory Configuration: $CALCULATED_MEMORY && JAVA_OPTS="$JAVA_OPTS $CALCULATED_MEMORY" && MALLOC_ARENA_MAX=2 SERVER_PORT=$PORT eval exec
                 $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher
     state     since                  cpu    memory        disk       logging               details
#0   running   2023-06-28T04:02:13Z   0.0%   46.5K of 1G   8K of 1G   1.2K/s of unlimited   

type:            task
sidecars:        
instances:       0/0
memory usage:    1024M
start command:   JAVA_OPTS="-agentpath:$PWD/.java-buildpack/open_jdk_jre/bin/jvmkill-1.17.0_RELEASE=printHeapHistogram=1 -Djava.io.tmpdir=$TMPDIR -XX:ActiveProcessorCount=$(nproc)
                 -Djava.ext.dirs=$PWD/.java-buildpack/container_security_provider:$PWD/.java-buildpack/open_jdk_jre/lib/ext -Djava.security.properties=$PWD/.java-buildpack/java_security/java.security $JAVA_OPTS" &&
                 CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-3.13.0_RELEASE -totMemory=$MEMORY_LIMIT -loadedClasses=22025 -poolType=metaspace -stackThreads=250
                 -vmOptions="$JAVA_OPTS") && echo JVM Memory Configuration: $CALCULATED_MEMORY && JAVA_OPTS="$JAVA_OPTS $CALCULATED_MEMORY" && MALLOC_ARENA_MAX=2 SERVER_PORT=$PORT eval exec
                 $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher
There are no running instances of this process.
```

<br>

# <div id="3"/>3.  Logging 서비스 관리

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
