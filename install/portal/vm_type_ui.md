### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Portal VM Type UI

## Table of Contents

1. [문서 개요](#1)  
    1.1. [목적](#1.1)  
    1.2. [범위](#1.2)  
    1.3. [참고자료](#1.3)  

2. [PaaS-TA AP Portal UI 설치](#2)  
    2.1. [Prerequisite](#2.1)   
    2.2. [Stemcell 확인](#2.2)    
    2.3. [Deployment 다운로드](#2.3)   
    2.4. [Deployment 파일 수정](#2.4)  
    2.5. [서비스 설치](#2.5)    
    2.6. [서비스 설치 확인](#2.6)  
    2.7. [Portal SSH 설치](#2.7)  

3. [PaaS-TA AP Portal 운영](#3)  
    3.1. [사용자의 조직 생성 Flag 활성화](#3.1)  
    3.2. [사용자포탈 UAA 페이지 오류](#3.2)  
    3.3. [운영자포탈 유저 페이지 조회 오류](#3.3)  
    3.4. [Log](#3.4)  
    3.5. [카탈로그 적용](#3.5)  
    3.6. [모니터링 및 오토스케일링 적용](#3.6)  


## <div id="1"/> 1. 문서 개요
### <div id="1.1"/> 1.1. 목적

본 문서(PaaS-TA AP Portal UI 설치 가이드)는 PaaS-TA AP Portal UI를 BOSH를 이용하여 설치 하는 방법을 기술하였다.

### <div id="1.2"/> 1.2. 범위
설치 범위는 PaaS-TA AP Portal을 검증하기 위한 Portal UI 기본 설치를 기준으로 작성하였다.

### <div id="1.3"/> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
Cloud Foundry Document: [https://docs.cloudfoundry.org](https://docs.cloudfoundry.org)  

## <div id="2"/> 2. PaaS-TA AP Portal UI 설치

### <div id="2.1"/> 2.1. Prerequisite  

본 설치 가이드는 Linux 환경에서 설치하는 것을 기준으로 하였다.  
서비스팩 설치를 위해서는 먼저 BOSH CLI v2 가 설치 되어 있어야 하고 BOSH 에 로그인이 되어 있어야 한다.<br>
BOSH CLI v2 가 설치 되어 있지 않을 경우 먼저 BOSH2.0 설치 가이드 문서를 참고 하여 BOSH CLI v2를 설치를 하고 사용법을 숙지 해야 한다.<br>
UAA client가 설치 되어 있지 않을 경우 UAA client의 설치가 필요하다.

- UAA client 설치 (BOSH Dependency 설치 필요)
```
$ sudo gem install cf-uaac
$ uaac -v
```

### <div id="2.2"/> 2.2. Stemcell 확인

Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-jammy 1.102를 사용한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                       Version   OS             CPI  CID  
bosh-openstack-kvm-ubuntu-jammy-go_agent  1.102      ubuntu-jammy  -    ce507ae4-aca6-4a6d-b7c7-220e3f4aaa7d

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
- PaaS-TA AP Portal UI에서 사용하는 변수는 system_domain, paasta_api_version, uaa_client_portal_secret, portal_web_user_language, portal_web_admin_language 이다.

> $ vi ~/workspace/common/common_vars.yml
```
... ((생략)) ...

system_domain: "61.252.53.246.nip.io"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
paasta_api_version: "v3"
uaa_client_portal_secret: "clientsecret"	# UAAC Portal Client에 접근하기 위한 Secret 변수
portal_web_user_language: ["ko", "en"]             # portal webuser language list (e.g. ["ko", "en"])
portal_web_admin_language: ["ko", "en"]             # portal webadmin language list (e.g. ["ko", "en"])
... ((생략)) ...
```



- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.

> $ vi ~/workspace/portal-deployment/portal-ui/vars.yml  
```
# STEMCELL INFO
stemcell_os: "ubuntu-jammy"                                             # stemcell os
stemcell_version: "1.102"                                                 # stemcell version

# NETWORKS INFO
private_networks_name: "default"                                         # private network name
public_networks_name: "vip"                                              # public network name

# MARIADB INFO
mariadb_azs: [z6]                                                        # mariadb : azs
mariadb_instances: 1                                                     # mariadb : instances (1)
mariadb_vm_type: "minimal"                                               # mariadb : vm type
mariadb_persistent_disk_type: "10GB"                                     # mariadb : persistent disk type
mariadb_port: "<MARIADB_PORT>"                                           # mariadb : database port (e.g. 13306) -- Do Not Use "3306"
mariadb_admin_password: "<MARIADB_ADMIN_PASSWORD>"                       # mariadb : database admin password (e.g. "mariadb")

# HAPROXY INFO
haproxy_azs: [z7]                                                        # haproxy : azs
haproxy_instances: 1                                                     # haproxy : instances (1)
haproxy_vm_type: "small"                                                 # haproxy : vm type
haproxy_public_ips: "<HAPROXY_PUBLIC_IPS>"                               # haproxy : public ips (e.g. "00.00.00.00")
haproxy_infra_admin: false                                               # haproxy : infra admin (default "false")

# PORTAL_WEBADMIN INFO
webadmin_azs: [z6]                                                       # webadmin : azs
webadmin_instances: 1                                                    # webadmin : instances (1)
webadmin_vm_type: "small"                                                # webadmin : vm type

# PORTAL_WEBUSER INFO
webuser_azs: [z6]                                                        # webuser : azs
webuser_instances: 1                                                     # webuser : instances (1)
webuser_vm_type: "small"                                                 # webuser : vm type
webuser_monitoring: false                                                # webuser : monitoring 사용 여부. true일 경우 앱 상세정보에서 모니터링창이 활성화 된다.
webuser_automaticapproval: false                                         # webuser : 회원가입시 PaaS-TA에 접속 가능 여부. true일 경우 관리자 포탈에서 승인을 해주어야 접근이 가능하다.
user_app_size: 0                                                         # webuser : 사용자 myApp 배포시 용량제한 여부 (값이 0일경우 무제한)

# ETC INFO
paasta_deployment_type: "ap"                                             # ETC : Paas-TA deployment type (ap, sidecar)
portal_default_api_name: "PaaS-TA"                                       # ETC : default api name (e.g. PaaS-TA {Version})
portal_default_api_url: "http://<PORTAL-API-HAPROXY-PUBLIC-IP>:2225"     # ETC : default api url
portal_default_header_auth: "Basic YWRtaW46b3BlbnBhYXN0YQ=="             # ETC : default header auth
portal_default_api_desc: "PaaS-TA infra"                                 # ETC : default api description (e.g. PaaS-TA {Version} infra)
apache_limit_request_body: <APACHE limitRequestBody>                     # Apache Limiting Upload File Size Directory / ex> 5000000
apache_usr_limit_request_body: <APACHE limitRequestBody>                 # Apache Limiting Upload File Size Directory webDir ex> 10240000
```

### <div id="2.5"/> 2.5. 서비스 설치

- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정하고, Option file을 추가할지 선택한다.  
     (선택) -o operations/cce.yml (CCE 조치를 적용하여 설치)

> $ vi ~/workspace/portal-deployment/portal-ui/deploy.sh
```
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"	# common_vars.yml File Path (e.g. ../../common/common_vars.yml)
CURRENT_IAAS="${CURRENT_IAAS}"			# IaaS Information (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 aws/azure/gcp/openstack/vsphere 입력)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"		# bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)

# DEPLOY
bosh -e ${BOSH_ENVIRONMENT} -n -d portal-ui deploy portal-ui.yml \
    -o operations/${CURRENT_IAAS}-network.yml \
    -o operations/cce.yml \
    -l ${COMMON_VARS_PATH} \
    -l vars.yml
```

- 서비스를 설치한다.  
```
$ cd ~/workspace/portal-deployment/portal-ui   
$ sh ./deploy.sh  
```

### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.  

> $ bosh -e ${BOSH_ENVIRONMENT} -d portal-ui vms  

```
Using environment '10.0.1.6' as client 'admin'

Task 4823. Done

Deployment 'portal-ui'

Instance                                                      Process State  AZ  IPs            VM CID                                   VM Type       Active  
haproxy/5c30c643-94d1-491c-9f6c-e72de4b0e6a4                  running        z7  10.30.56.10    vm-891ff2dd-4ee0-4c42-8fa8-b2d0cf0b8537  portal_tiny   true  
									         115.68.46.180                                                           
mariadb/19bf81a9-cde9-432b-87ca-cbac1f28854a                  running        z6  10.30.56.9     vm-7a6f8042-e9b8-434c-abbf-776bbfd3386d  portal_small  true  
paas-ta-portal-webadmin/bc536f61-10bd-4702-af5f-5e63500e110e  running        z6  10.30.56.11    vm-176ccac5-f154-4420-b821-9ed30a18f3e2  portal_small  true  
paas-ta-portal-webuser/409c038b-d013-41d3-b6b2-aebb4a02d908   running        z6  10.30.56.12    vm-d9cf481f-64c7-45fd-aadb-e4eb1b31945a  portal_tiny   true  

4 vms

Succeeded
```

### <div id="2.7"/> 2.7. Portal SSH 설치

Portal 5.1.0 버전 이상부터는 배포된 어플리케이션의 SSH 접속이 가능하다.

이를 위해 Portal SSH App을 먼저 배포해야 한다.

- Portal 배포를 위한 조직 및 공간 생성
```
### Portal 배포를 위한 조직 및 공간 생성 및 설정
$ cf create-quota portal_quota -m 20G -i -1 -s -1 -r -1 --reserved-route-ports -1 --allow-paid-service-plans
$ cf create-org portal -q portal_quota
$ cf create-space system -o portal  
$ cf target -o portal -s system
```


- Portal SSH 다운로드 및 배포
```
$ cd ~/workspace/portal-deployment
$ wget --content-disposition https://nextcloud.paas-ta.org/index.php/s/awPjYDYCMiHY7yF/download
$ unzip portal-ssh.zip
$ cd portal-ssh
$ cf push
```


## <div id="3"/>3. PaaS-TA AP Portal 운영

### <div id="3.1"/> 3.1. 사용자의 조직 생성 Flag 활성화

PaaS-TA는 기본적으로 일반 사용자는 조직을 생성할 수 없도록 설정되어 있다. 포털 배포를 위해 조직 및 공간을 생성해야 하고 또 테스트를 구동하기 위해서도 필요하므로 사용자가 조직을 생성할 수 있도록 user_org_creation FLAG를 활성화 한다. FLAG 활성화를 위해서는 PaaS-TA 운영자 계정으로 로그인이 필요하다.

```
$ cf enable-feature-flag user_org_creation
```
```
Setting status of user_org_creation as admin...
OK

Feature user_org_creation Enabled.
```

### <div id="3.2"/> 3.2. 사용자포탈 UAA 페이지 오류
>![paas-ta-portal-31]
1. uaac portalclient가 등록이 되어있지 않다면 해당 화면과 같이 redirect오류가 발생한다.
2. uaac client add를 통해 potalclient를 추가시켜주어야 한다.
    > $ uaac target\
    $ uaac token client get\
        Client ID:  admin\
        Client secret:  *****

3. uaac client add portalclient –s “portalclient Secret”
>--redirect_uri "사용자포탈 Url, 사용자포탈 Url/callback"\
$ uaac client add portalclient -s xxxxx --redirect_uri "http://portal-web-user.xxxx.nip.io, http://portal-web-user.xxxx.nip.io/callback" \
--scope "cloud_controller_service_permissions.read , openid , cloud_controller.read , cloud_controller.write , cloud_controller.admin" \
--authorized_grant_types "authorization_code , client_credentials , refresh_token" \
--authorities="uaa.resource" \
--autoapprove="openid , cloud_controller_service_permissions.read"

 >![paas-ta-portal-32]
1. uaac portalclient가 url이 잘못 등록되어있다면 해당 화면과 같이 redirect오류가 발생한다.
2. uaac client update를 통해 url을 수정해야한다.
   > $ uaac target\
    $ uaac token client get\
   Client ID:  admin\
   Client secret:  *****
3. uaac client update portalclient --redirect_uri "사용자포탈 Url, 사용자포탈 Url/callback"
    
    >$ uaac client update portalclient --redirect_uri "http://portal-web-user.xxxx.nip.io, http://portal-web-user.xxxx.nip.io/callback"

### <div id="3.3"/> 3.3. 운영자 포탈 유저 페이지 조회 오류
1. 페이지 이동시 정보를 가져오지 못하고 오류가 났을 경우 common-api VM으로 이동후에 DB 정보 config를 수정후 재시작을 해 주어야 한다.

### <div id="3.4"/> 3.4. Log
Paas-TA Portal 각각 Instance의 log를 확인 할 수 있다.
1. 로그를 확인할 Instance에 접근한다.
    > bosh ssh -d [deployment name] [instance name]

       Instance                                                          Process State  AZ  IPs            VM CID                                   VM Type        Active   
       haproxy/8cc2d633-2b43-4f3d-a2e8-72f5279c11d5                      running        z5  10.30.107.213  vm-315bfa1b-9829-46de-a19d-3bd65e9f9ad4  portal_large   true  
                                                                                            115.68.46.214                                                            
       mariadb/117cbf05-b223-4133-bf61-e15f16494e21                      running        z5  10.30.107.211  vm-bc5ae334-12d4-41d4-8411-d9315a96a305  portal_large   true  
       paas-ta-portal-webadmin/8047fcbd-9a98-4b61-b161-0cbb277fa643      running        z5  10.30.107.221  vm-188250fd-e918-4aab-9cbe-7d368852ea8a  portal_medium  true  
       paas-ta-portal-webuser/cb206717-81c9-49ed-a0a8-e6c3b957cb66       running        z5  10.30.107.222  vm-822f68a5-91c8-453a-b9b3-c1bbb388e377  portal_medium  true  

       11 vms

       Succeeded
       inception@inception:~$ bosh ssh -d paas-ta-portal-ui paas-ta-portal-webadmin  << instance 접근(bosh ssh) 명령어 입력
       Using environment '10.30.40.111' as user 'admin' (openid, bosh.admin)

       Using deployment 'paas-ta-portal-webadmin'

       Task 5195. Done
       Unauthorized use is strictly prohibited. All access and activity
       is subject to logging and monitoring.
       Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 4.4.0-92-generic x86_64)

        * Documentation:  https://help.ubuntu.com/

       The programs included with the Ubuntu system are free software;
       the exact distribution terms for each program are described in the
       individual files in /usr/share/doc/*/copyright.

       Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
       applicable law.

       Last login: Tue Sep  4 07:11:42 2018 from 10.30.20.28
       To run a command as administrator (user "root"), use "sudo <command>".
       See "man sudo_root" for details.

       paas-ta-portal-webadmin/48fa0c5a-52eb-4ae8-a7b9-91275615318c:~$

2. 로그파일이 있는 폴더로 이동한다.
    > 위치 : /var/vcap/sys/log/[job name]/

         paas-ta-portal-webadmin/48fa0c5a-52eb-4ae8-a7b9-91275615318c:~$ cd /var/vcap/sys/log/paas-ta-portal-webadmin/
         paas-ta-portal-webadmin/48fa0c5a-52eb-4ae8-a7b9-91275615318c:/var/vcap/sys/log/paas-ta-portal-webadmin$ ls
         paas-ta-portal-webadmin.stderr.log  paas-ta-portal-webadmin.stdout.log

3. 로그파일을 열어 내용을 확인한다.
    > vim [job name].stdout.log

        예)
        vim paas-ta-portal-webadmin.stdout.log
        2018-09-04 02:08:42.447 ERROR 7268 --- [nio-2222-exec-1] p.p.a.e.GlobalControllerExceptionHandler : Error message : Response : org.springframework.security.web.firewall.FirewalledResponse@298a1dc2
        Occured an exception : 403 Access token denied.
        Caused by...
        org.cloudfoundry.client.lib.CloudFoundryException: 403 Access token denied. (error="access_denied", error_description="Access token denied.")
                at org.cloudfoundry.client.lib.oauth2.OauthClient.createToken(OauthClient.java:114)
                at org.cloudfoundry.client.lib.oauth2.OauthClient.init(OauthClient.java:70)
                at org.cloudfoundry.client.lib.rest.CloudControllerClientImpl.initialize(CloudControllerClientImpl.java:187)
                at org.cloudfoundry.client.lib.rest.CloudControllerClientImpl.<init>(CloudControllerClientImpl.java:163)
                at org.cloudfoundry.client.lib.rest.CloudControllerClientFactory.newCloudController(CloudControllerClientFactory.java:69)
                at org.cloudfoundry.client.lib.CloudFoundryClient.<init>(CloudFoundryClient.java:138)
                at org.cloudfoundry.client.lib.CloudFoundryClient.<init>(CloudFoundryClient.java:102)
                at org.openpaas.paasta.portal.api.service.LoginService.login(LoginService.java:47)
                at org.openpaas.paasta.portal.api.controller.LoginController.login(LoginController.java:51)
                at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
                at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
                at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
                at java.lang.reflect.Method.invoke(Method.java:498)
                at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:205)
                at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:133)
                at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:97)
                at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:827)
                at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:738)
                at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
                at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:967)
                at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:901)
                at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
                at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:872)
                at javax.servlet.http.HttpServlet.service(HttpServlet.java:661)
                at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
                at javax.servlet.http.HttpServlet.service(HttpServlet.java:742)
                at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
                at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
                at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
                at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
                at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)

### <div id="3.5"/> 3.5. 카탈로그 적용
##### 1. Catalog 빌드팩, 서비스팩 추가
Paas-TA Portal 설치 후에 관리자 포탈에서 빌드팩, 서비스팩을 등록해야 사용자 포탈에서 사용이 가능하다.
- [카탈로그 이미지 다운로드](https://nextcloud.paas-ta.org/index.php/s/EmzfJw38H4GQKTr/download)

 1. 관리자 포탈에 접속한다.(portal-web-admin.[public ip].nip.io)
    
    >![paas-ta-portal-15]
 2. 운영관리를 누른다.
    
    >![paas-ta-portal-16]
 3. 카탈로그 페이지에 들어간다.
    
    >![paas-ta-portal-17]
 4. 빌드팩, 서비스팩 상세화면에 들어가서 각 항목란에 값을 입력후에 저장을 누른다.
    
    >![paas-ta-portal-18]

    ※ 카탈로그 등록 및 수정 시 카탈로그 관리 코드는 선택 필수이며, 현재 사용 가능한 코드가 없는 경우 다음 내용을 참고하여 처리하도록 한다.
    1. ①"코드 관리"를 클릭한다.
    2. **Group Table**에서 해당하는 ②"분류 코드"를 클릭한다.
    3. **Detail Table**에 ③"등록"버튼을 클릭하여 카탈로그 관리 코드를 추가 후 사용한다.
    ![paas-ta-portal-18-1]
 5. 사용자포탈에서 변경된값이 적용되어있는지 확인한다.
    
    >![paas-ta-portal-19]

### <div id="3.6"/> 3.6. 모니터링 및 오토스케일링 적용
##### 1. 포탈 설치 이전 모니터링 설정 적용
###### 1.PaaS-TA 에서 제공하고있는 모니터링을 미리 설치를 한 후에 진행해야 한다.
 1. Paas-TA Portal 설치 시 공통 변수 파일과 Deployment 변수 파일의 monitoring_api_url=<모니터링 API URL>, webuser_monitoring=true로 적용 한 후 설치 하면 정상적으로 모니터링 페이지 및 오토스케일링을 사용 할 수 있다.

##### 2. 포탈 설치 이후 모니터링 설정 적용
 1. 사용자 포탈의 앱 상세 페이지로 이동한다.
    
    >![paas-ta-portal-30]
 2. ① 상세페이지 레이아웃 하단의 모니터링 버튼을 누른다.

 3. ② 모니터링 오토 스케일링 화면

 4. ③ 모니터링 알람 설정 화면

 5. 추이차트 탭에서 디스크 메모리 네트워크 사용량을 인스턴스 별로 확인이 가능하다.        

[paas-ta-portal-01]:./images/Paas-TA-Portal_01.png
[paas-ta-portal-02]:./images/Paas-TA-Portal_02.png
[paas-ta-portal-03]:./images/Paas-TA-Portal_03.png
[paas-ta-portal-04]:./images/Paas-TA-Portal_04.png
[paas-ta-portal-05]:./images/Paas-TA-Portal_05.png
[paas-ta-portal-06]:./images/Paas-TA-Portal_06.png
[paas-ta-portal-07]:./images/Paas-TA-Portal_07.png
[paas-ta-portal-08]:./images/Paas-TA-Portal_08.png
[paas-ta-portal-09]:./images/Paas-TA-Portal_09.png
[paas-ta-portal-10]:./images/Paas-TA-Portal_10.png
[paas-ta-portal-11]:./images/Paas-TA-Portal_11.png
[paas-ta-portal-12]:./images/Paas-TA-Portal_12.png
[paas-ta-portal-13]:./images/Paas-TA-Portal_13.png
[paas-ta-portal-14]:./images/Paas-TA-Portal_14.png
[paas-ta-portal-15]:./images/Paas-TA-Portal_15.png
[paas-ta-portal-16]:./images/Paas-TA-Portal_16.png
[paas-ta-portal-17]:./images/Paas-TA-Portal_17.png
[paas-ta-portal-18]:./images/Paas-TA-Portal_18.png
[paas-ta-portal-18-1]:./images/Paas-TA-Portal_18-1.png
[paas-ta-portal-19]:./images/Paas-TA-Portal_19.png
[paas-ta-portal-20]:./images/Paas-TA-Portal_20.png
[paas-ta-portal-21]:./images/Paas-TA-Portal_21.png
[paas-ta-portal-22]:./images/Paas-TA-Portal_22.png
[paas-ta-portal-23]:./images/Paas-TA-Portal_23.png
[paas-ta-portal-24]:./images/Paas-TA-Portal_24.png
[paas-ta-portal-25]:./images/Paas-TA-Portal_25.png
[paas-ta-portal-26]:./images/Paas-TA-Portal_26.png
[paas-ta-portal-27]:./images/Paas-TA-Portal_27.PNG
[paas-ta-portal-28]:./images/Paas-TA-Portal_28.PNG
[paas-ta-portal-29]:./images/Paas-TA-Portal_29.png
[paas-ta-portal-30]:./images/Paas-TA-Portal_30.png
[paas-ta-portal-31]:./images/Paas-TA-Portal_27.jpg
[paas-ta-portal-32]:./images/Paas-TA-Portal_28.jpg


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Install](../README.md) > Portal VM Type UI
