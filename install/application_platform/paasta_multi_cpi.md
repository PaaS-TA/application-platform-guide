### [Index](https://github.com/okpc579/paasta-guide-new/blob/main/README.md) > [AP Install](../README.md) > PaaS-TA Multi CPI

## Table of Contents

1. [개요](#1)  
 1.1. [목적](#1.1)  
 1.2. [범위](#1.2)  
 1.3. [참고 자료](#1.3)  
2. [Multi CPI](#2)  
 2.1. [Prerequisite](#2.1)  
 2.2. [설치 파일 다운로드](#2.2)  
 2.3. [OpenVPN](#2.3)  
　2.3.1. [변수 설정](#2.3.1)       
　2.3.2. [인증서 생성](#2.3.2)       
　2.3.3. [OpenVPN 설치](#2.3.3)  
　2.3.4. [OpenVPN 연결 확인](#2.3.4)  
　2.3.5. [정적 라우팅 추가 (선택)](#2.3.5)  
 2.4. [Multi CPI 설정](#2.4)   
　2.4.1. [BOSH 설치](#2.4.1)    
　2.4.2. [CPI Config 설정](#2.4.2)    
　　2.4.2.1. [Same IaaS AZ의 경우](#2.4.2.1)    
　　2.4.2.2. [Different IaaS AZ의 경우](#2.4.2.2)    
　2.4.3. [Cloud Config 설정](#2.4.3)    
　　2.4.3.1. [Same IaaS AZ의 경우](#2.4.3.1)    
　　2.4.3.2. [Different IaaS AZ의 경우](#2.4.3.2)    
　2.4.4. [Stemcell 업로드](#2.4.4)    
　2.4.5. [Multi CPI를 이용한 AP 설치 테스트](#2.4.5)    
 
# <div id='1'/>1.  문서 개요

## <div id='1.1'/>1.1. 목적
본 문서는 BOSH2(이하 BOSH)의 Multi CPI 설정 가이드 문서로, 하나의 BOSH를 통하여 BOSH가 설치된 IaaS 환경(이하 Main IaaS AZ)과 BOSH가 설치되지 않은 다른 IaaS 환경(이하 Second IaaS AZ)에서 VM을 배포하는 Multi CPI를 설정하고 사용하는 방법에 관해서 설명하였다.  

<br>

## <div id='1.2'/>1.2. 범위
본 가이드는 BOSH와 PaaS-TA AP에 대한 기본 이해도가 있다는 전제 하에 가이드를 진행하였다.  
multi-cpi-deployment는 paasta-deployment v5.6.5의 설치를 기준으로 가이드를 작성하였다.  
multi-cpi-deployment는 AWS, OpenStack, vSphere 에서 설정이 가능하다.  
분류는 크게 Main IaaS AZ와 Second IaaS AZ가 같은 경우 (e.g. A OpenStack ⇔ B OpenStack, 이하 Same IaaS AZ) 와 Main IaaS AZ와 Second IaaS AZ가 다른 경우 (e.g. Openstack ⇔ AWS, 이하 Different IaaS AZ)를 기준으로 작성하였다.

설정 가능한 IaaS 케이스는 다음과 같다.  


<table class="tg">
<thead>
  <tr>
    <th class="tg-9wq8">Category</th>
    <th class="tg-9wq8">Main IaaS AZ</th>
    <th class="tg-9wq8">Second IaaS AZ</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-9wq8" rowspan="6">Different IaaS AZ</td>
    <td class="tg-0pky">AWS</td>
    <td class="tg-0pky">OpenStack</td>
  </tr>
  <tr>
    <td class="tg-0pky">AWS</td>
    <td class="tg-0pky">vSphere</td>
  </tr>
  <tr>
    <td class="tg-0pky">OpenStack</td>
    <td class="tg-0pky">AWS</td>
  </tr>
  <tr>
    <td class="tg-0pky">OpenStack</td>
    <td class="tg-0pky">vSphere</td>
  </tr>
  <tr>
    <td class="tg-0lax">vSphere</td>
    <td class="tg-0lax">AWS</td>
  </tr>
  <tr>
    <td class="tg-0lax">vSphere</td>
    <td class="tg-0lax">OpenStack</td>
  </tr>
  <tr>
    <td class="tg-nrix" rowspan="3">Same IaaS AZ</td>
    <td class="tg-0lax">AWS</td>
    <td class="tg-0lax">AWS</td>
  </tr>
  <tr>
    <td class="tg-0lax">OpenStack</td>
    <td class="tg-0lax">OpenStack</td>
  </tr>
  <tr>
    <td class="tg-0lax">vSphere</td>
    <td class="tg-0lax">vSphere</td>
  </tr>
</tbody>
</table>
   
<br>

## <div id='1.3'/>1.3. 참고 자료

본 문서는 Cloud Foundry의 BOSH Document와 cf-deployment, Open VPN을 참고로 작성하였다.

BOSH Document: [https://bosh.io](https://bosh.io)  
CF Document: [https://docs.cloudfoundry.org/](https://docs.cloudfoundry.org/)  
BOSH cpi-config Document: [https://bosh.io/docs/cpi-config](https://bosh.io/docs/cpi-config)  
BOSH guide-multi-cpi-aws Document: [https://bosh.io/docs/guide-multi-cpi-aws](https://bosh.io/docs/guide-multi-cpi-aws)  
BOSH Deployment: [https://github.com/cloudfoundry/bosh-deployment](https://github.com/cloudfoundry/bosh-deployment)  
CF Deployment: [https://github.com/cloudfoundry/cf-deployment](https://github.com/cloudfoundry/cf-deployment)  
OpenVPN : [https://openvpn.net/](https://openvpn.net/)  

<br><br>

# <div id='2'/>2.  Multi CPI
BOSH에 Multi CPI를 설정할 경우 하나의 BOSH를 통하여 Main IaaS AZ와 Second IaaS AZ 두개의 환경에서 VM을 각각 배포할 수 있다.  
본 가이드에서는 Main IaaS AZ와 Second IaaS AZ에 OpenVPN을 각각 설치한 후, Main IaaS AZ에 BOSH를 설치한 뒤 Multi CPI 설정을 진행한다.  

<br>

## <div id='2.1'/>2.1. Prerequisite

본 가이드는 Linux 환경에서 진행하는 것을 기준으로 하였다.  
또한 Multi CPI 설정를 위해서는 먼저 BOSH CLI가 설치 되어 있어야 한다.  
BOSH CLI가 설치 되어 있지 않을 경우 먼저 BOSH 설치 가이드 문서를 참고 하여 BOSH CLI를 설치를 진행 한다.


<br>


## <div id='2.2'/>2.2. 설치 파일 다운로드

- BOSH를 설치하기 위한 paasta-deployment와 Multi CPI 설정을 위한 multi-cpi-deployment가 존재하지 않는다면 다운로드 받는다

```
$ mkdir -p ~/workspace
$ cd ~/workspace
$ git clone https://github.com/PaaS-TA/paasta-deployment.git -b v5.6.5
$ git clone https://github.com/PaaS-TA/multi-cpi-deployment.git -b v5.6.2
```

<br>

## <div id='2.3'/>2.3. OpenVPN
BOSH가 Main IaaS AZ와 Second IaaS AZ의 통신을 진행하기 위하여 OpenVPN을 Main IaaS AZ와 Second IaaS AZ에  설치를 진행한다.

<br>

### <div id='2.3.1'/>2.3.1. 변수 설정
```
# OpenVPN 설치를 위한 deployment 폴더 이동
$ cd ~/workspace/multi-cpi-deployment/openvpn
```

- Main IaaS AZ에 설치되는 OpenVPN az1 변수를 설정한다.  
  (OpenVPN을 설치할 IaaS 환경에 대한 주석을 해제하고 변수를 설정한다.)
> $ vi vars-az1.yml
```
### openvpn default
wan_ip: "XXX.XXX.XXX.XXX"                         # Used by OpenVPN Server-1 ip 
push_routes: ["XXX.XXX.XXX.XXX 255.255.255.0"]    # ex) 10.0.10.0 255.255.255.0
lan_gateway: "XXX.XXX.XXX.XXX"                    # ex) 10.0.10.1
lan_ip: "XXX.XXX.XXX.XXX"                         # ex) 10.0.10.10
lan_network: "XXX.XXX.XXX.XXX"                    # ex) 10.0.10.0
lan_network_mask_bits: "24"                       # ex) 24
vpn_network: "XXX.XXX.XXX.XXX"                    # ex) 192.168.0.0 (az-1: 192.168.0.0, az-2: 192.168.1.0)
vpn_network_mask: "XXX.XXX.XXX.XXX"               # ex) 255.255.255.0
vpn_network_mask_bits: "24"                       # ex) 24
remote_network_cidr_block: "XXX.XXX.XXX.XXX/24"   # ex) 20.0.20.0/24
remote_vpn_ip: "XXX.XXX.XXX.XXX"                  # Used by OpenVPN Server-2 ip 

#### IaaS :: AWS
#access_key_id: "XXXXXXXXXXXXXXX"                  # AWS Access Key
#secret_access_key: "XXXXXXXXXXXXXXX"              # AWS Secret Key
#region: "ap-northeast-2"                          # AWS Region
#availability_zone: "ap-northeast-2a"              # AWS Region
#subnet_id: "paasta-subnet"                        # AWS Subnet ex) subnet-0ebc.....
#default_security_groups: ["bosh-sg"]              # AWS Security-Group
#bootstrap_ssh_key_name: "bosh-key"                # AWS SSH Private Key Name
#bootstrap_ssh_key_path: "/.ssh/bosh-key.pem"      # AWS SSH Private Key Path
#route_table_id: "XXXXXXXXXXXXXXX"                 # AWS Route table id ex) rtb-4127..

#### IaaS :: openstack 
#auth_url: "http://XXX.XXX.XXX.XXX:5000/v3/"       # Openstack Keystone URL
#az: "nova"                                        # Openstack AZ Zone
#default_key_name: "bosh-key"                      # Openstack Key Name
#default_security_groups: ["bosh-sg"]              # Openstack Security Group
#net_id: "XXXXXXXXXXXXXXX"                         # Openstack Network ID ex) 70f151db-274....
#openstack_password: "XXXXXXXXXXXXXXX"             # Openstack User Password
#openstack_username: "XXXXXXXXXXXXXXX"             # Openstack User Name
#openstack_domain: "default"                       # Openstack Domain Name
#openstack_project: "paasta"                       # Openstack Project
#private_key: "/.ssh/bosh-key.pem"                 # Openstack SSH Private Key Path
#region: "RegionOne"                               # Openstack Region

#### IaaS :: vsphere 
#wan_gateway: "XXX.XXX.XXX.XXX"                    # Used by OpenVPN gateway ex) 172.100.100.100
#wan_network: "XXX.XXX.XXX.XXX"                    # Used by OpenVPN Network ex) 172.100.100.0
#wan_network_mask_bits: "24"                       # Used by OpenVPN Mask bits ex) 24
#wan_network_name: "Public Network"                # vCenter Public Network Name 
#lan_network_name: "Private Network"               # vCenter Private Network Name 
#vcenter_dc: "DataCenter"                          # vCenter Data Center Name
#vcenter_cluster: "CLUSTER"                        # vCenter Cluster Name
#vcenter_ip: "XXX.XXX.XXX.XXX"                     # vCenter Private IP
#vcenter_user: "XXXXXXXXXXXXXXX"                   # vCenter User Name
#vcenter_password: "XXXXXXXXXXXXXXX"               # vCenter User Password
#vcenter_ds: "DataStorage"                         # vCenter Data Storage Name
#vcenter_templates: "Templates"                    # vCenter Templates Name
#vcenter_vms: "Vms"                                # vCenter VMS Name
#vcenter_disks: "Disks"                            # vCenter Disk Name
```

- Second IaaS AZ에 설치되는 OpenVPN az2 변수를 설정한다.  
  (OpenVPN을 설치할 IaaS 환경에 대한 주석을 해제하고 변수를 설정한다.)
> $ vi vars-az2.yml
```
### openvpn default
wan_ip: "XXX.XXX.XXX.XXX"                         # Used by OpenVPN Server-2 ip 
push_routes: ["XXX.XXX.XXX.XXX 255.255.255.0"]    # ex) 20.0.20.0 255.255.255.0
lan_gateway: "XXX.XXX.XXX.XXX"                    # ex) 20.0.20.1
lan_ip: "XXX.XXX.XXX.XXX"                         # ex) 20.0.20.10
lan_network: "XXX.XXX.XXX.XXX"                    # ex) 20.0.20.0
lan_network_mask_bits: "24"                       # ex) 24
vpn_network: "XXX.XXX.XXX.XXX"                    # ex) 192.168.1.0 (az-1: 192.168.0.0, az-2: 192.168.1.0)
vpn_network_mask: "XXX.XXX.XXX.XXX"               # ex) 255.255.255.0
vpn_network_mask_bits: "24"                       # ex) 24
remote_network_cidr_block: "XXX.XXX.XXX.XXX/24"   # ex) 10.0.10.0/24
remote_vpn_ip: "XXX.XXX.XXX.XXX"                  # Used by OpenVPN Server-1 ip 

... ((생략)) ...

```

<br>

### <div id='2.3.2'/>2.3.2. 인증서 생성
OpenVPN에서 사용 할 인증서를 generate_ca.sh을 실행하여 생성한다.
```
$ source generate_ca.sh

$ ll creds
drwxrwxr-x 2 ubuntu ubuntu  4096 Jul 22 02:12 ./
drwxrwxr-x 4 ubuntu ubuntu  4096 Jul 21 01:47 ../
-rw-rw-r-- 1 ubuntu ubuntu 18098 Jul 14 04:22 vpn-server-az1.yml
-rw-rw-r-- 1 ubuntu ubuntu 18102 Jul 14 04:22 vpn-server-az2.yml
```

<br>

### <div id='2.3.3'/>2.3.3. OpenVPN 설치
- deploy-vpn-\*.sh에 사용할 IaaS에 대한 옵션을 추가한다.
```
### OpenVPN az1 IaaS 설정
$ vi deploy-vpn-az1.sh

ex) Choose one and add it.
-o operations/init-aws.yml \
-o operations/init-openstack.yml \
-o operations/init-vsphere.yml \


### OpenVPN az2 IaaS 설정 
$ vi deploy-vpn-az2.sh

ex) Choose one and add it.
-o operations/init-aws.yml \
-o operations/init-openstack.yml \
-o operations/init-vsphere.yml \
```


- 생성된 인증서를 사용하여 OpenVPN az1, OpenVPN az2 설치를 진행한다.
```
### OpenVPN az1 설치
$ source deploy-vpn-az1.sh

### OpenVPN az2 설치
$ source deploy-vpn-az2.sh
```

<br>

### <div id='2.3.4'/>2.3.4. OpenVPN 연결 확인
OpenVPN이 설치가 완료되면 상호간 연결이 가능한지 확인한다.
```
# OpenVPN 인증키 생성
## OpenVPN az1.key
$ bosh int creds/vpn-deploy-az1.yml --path /ssh/private_key > openvpn-az1.key 
$ chmod 600 openvpn-az1.key

## OpenVPN az2.key
$ bosh int creds/vpn-deploy-az2.yml --path /ssh/private_key > openvpn-az2.key 
$ chmod 600 openvpn-az2.key


# 연결 확인
## openVPN az1
$ ssh openvpn@{openvpn-az1-ip} -i openvpn-az1.key 

## ping 연결 확인
$ ping {openvpn-az2-ip}

## ifconfig 확인 (네트워크 인터페이스 tun0, tun2 확인)
$ sudo su
$ ifconfig 
```

<br>

### <div id='2.3.5'/>2.3.5. 정적 라우팅 추가 (선택)
클라이언트 터널을 사용하기 위하여 정적 라우팅을 추가할 수 있다.  
IaaS에서 정적 경로 추가를 지원하지 않는 경우 OpenVPN 의 네트워크를 사용할 모든 VM에서 해당 명령어를 통하여 정적 라우팅을 설정한다.
```
$ sudo ip route add {remote_network_cidr_block} via {lan_ip}
e.g.) sudo ip route add 20.0.20.0/24 via 10.0.10.10

$ ping {remote_network_ip}
```

<br>

## <div id='2.4'/>2.4. Multi CPI 설정
BOSH를 설치하고 Multi CPI를 설정하여 하나의 BOSH로 Main IaaS AZ와 Second IaaS AZ에서 VM을 배포할 수 있다.  

- Multi CPI 파일 BOSH 폴더로 이동
```
$ cp ~/workspace/multi-cpi-deployment/multi-cpi ~/workspace/paasta-deployment/bosh -r
$ cd ~/workspace/paasta-deployment/bosh
```

<br>

### <div id='2.4.1'/>2.4.1. BOSH 설치
Same IaaS AZ의 경우 다른 옵션을 추가하지 않고 BOSH를 설치하며, Different IaaS AZ의 경우 옵션을 추가하여 BOSH 설치를 진행한다.    
본 가이드에서는 추가되는 옵션에대한 설명을 진행한다.  
본 가이드에서는 4개의 예제를 기술했지만 상황에 맞춰서 옵션과 배포파일을 변경하여 진행한다.  
BOSH 설치에 대한 상세 내용은 BOSH 설치 가이드를 참고한다.

- Multi CPI 관련 Option 파일

|파일명|설명|
|------|---|
| deploy-cpi-aws-secondary.yml | BOSH를 설치하지 않는 인프라가 AWS 일 경우 사용 |
| deploy-cpi-openstack-secondary.yml	 | BOSH를 설치하지 않는 인프라가 OpenStack 일 경우 사용 |
| deploy-cpi-vsphere-secondary.yml	 | BOSH를 설치하지 않는 인프라가 vSphere 일 경우 사용 |
| deploy-cpi-registry-secondary.yml | BOSH를 설치하는 인프라가 vSphere 일 경우 사용 |

- 예제1. AWS - Openstack BOSH 설치
> $ vi deploy-aws.sh
```diff
 bosh create-env bosh.yml \
 	--state=aws/state.json \
 	--vars-store=aws/creds.yml \
 	-o aws/cpi.yml \
+ 	-o multi-cpi/deploy-cpi-openstack-secondary.yml \
 	-o uaa.yml \
 	-o credhub.yml \
 	-o jumpbox-user.yml \
 	-o cce.yml \
 	-l aws-vars.yml
```

- 예제2. AWS - vSphere BOSH 설치
> $ vi deploy-aws.sh
```diff
 bosh create-env bosh.yml \
 	--state=aws/state.json \
 	--vars-store=aws/creds.yml \
 	-o aws/cpi.yml \
+ 	-o multi-cpi/deploy-cpi-vsphere-secondary.yml \
 	-o uaa.yml \
 	-o credhub.yml \
 	-o jumpbox-user.yml \
 	-o cce.yml \
 	-l aws-vars.yml
```


- 예제3. Openstack - vSphere BOSH 설치
> $ vi deploy-openstack.sh
```diff
 bosh create-env bosh.yml \
 	--state=openstack/state.json \
 	--vars-store=openstack/creds.yml \
 	-o openstack/cpi.yml \
+ 	-o multi-cpi/deploy-cpi-vsphere-secondary.yml \
 	-o uaa.yml \
 	-o credhub.yml \
 	-o jumpbox-user.yml \
 	-o cce.yml \
 	-o openstack/disable-readable-vm-names.yml \
 	-l openstack-vars.yml
```

- 예제4. vSphere - AWS BOSH 설치
> $ vi deploy-vsphere.sh
```diff
 bosh create-env bosh.yml \
 	--state=vsphere/state.json \
 	--vars-store=vsphere/creds.yml \
 	-o vsphere/cpi.yml \
 	-o vsphere/resource-pool.yml  \
+ 	-o multi-cpi/deploy-cpi-aws-secondary.yml \
+ 	-o multi-cpi/deploy-cpi-registry-secondary.yml \
 	-o uaa.yml  \
 	-o credhub.yml  \
 	-o jumpbox-user.yml  \
 	-l vsphere-vars.yml
```


- 변수 설정 후 BOSH 설치 진행
```
$ vi {IaaS}-vars.yml
$ source deploy-{IaaS}.sh
```

<br>


### <div id='2.4.2'/>2.4.2. CPI Config 설정
CPI에 대한 추가 설정을 진행한다.  
해당되는 IaaS에 맞게 cpi-config.yml의 주석을 해제하여 진행한다.
|파일명|설명|
|------|---|
| cpi-config.yml	 | multi-cpi 추가를 위한 cpi config file |
| cpi-vars.yml	 | multi-cpi 설정 파일 |

#### <div id='2.4.2.1'/>2.4.2.1. Same IaaS AZ의 경우

- 예제 AWS - AWS를 사용할 경우
> $ vi multi-cpi/cpi-vars.yml
```
... ((생략)) ...

## MULTI-CPI VARIABLE :: AWS
aws_access_key_id: "XXXXXXXXXXXXXXX"                    # AWS Access Key
aws_secret_access_key: "XXXXXXXXXXXXX"                  # AWS Secret Key
aws_default_key_name: "paasta-key"                      # AWS Key Name
aws_default_security_groups: ["paasta-security"]        # AWS Security-Group
aws_region: "ap-northeast-2"                            # AWS Region

... ((생략)) ...

# IF USE SAME IAAS, CPI MULTI-CPI VARIABLE

## MULTI-CPI VARIABLE :: AWS second
aws_second_access_key_id: "XXXXXXXXXXXXXXX"                    # AWS Second Access Key
aws_second_secret_access_key: "XXXXXXXXXXXXX"                  # AWS Second Secret Key
aws_second_default_key_name: "paasta-key"                      # AWS Second Key Name
aws_second_default_security_groups: ["paasta-security"]        # AWS Second Security-Group
aws_second_region: "ap-northeast-2"                            # AWS Second Region

... ((생략)) ...
```

> $ vi multi-cpi/cpi-config.yml (사용할 IaaS 정보를 주석 해제한다.)
```
#### DIFFERENT IAAS CPI

cpis:
- name: aws-cpi
  type: aws
  properties:
    access_key_id: ((aws_access_key_id))
    secret_access_key: ((aws_secret_access_key))
    default_key_name: ((aws_default_key_name))
    default_security_groups: ((aws_default_security_groups))
    region: ((aws_region))

... ((생략)) ...

#### SAME IAAS CPI

- name: aws-cpi-second
  type: aws
  properties:
    access_key_id: ((aws_second_access_key_id))
    secret_access_key: ((aws_second_secret_access_key))
    default_key_name: ((aws_second_default_key_name))
    default_security_groups: ((aws_second_default_security_groups))
    region: ((aws_second_region))

... ((생략)) ...
```

- CPI Config 적용
```
$ bosh update-cpi-config multi-cpi/cpi-config.yml -l multi-cpi/cpi-vars.yml
```

#### <div id='2.4.2.2'/>2.4.2.2. Different IaaS AZ의 경우
- 예제 AWS - OpenStack을 사용할 경우
> $ vi multi-cpi/cpi-vars.yml
```
... ((생략)) ...

## MULTI-CPI VARIABLE :: AWS
aws_access_key_id: "XXXXXXXXXXXXXXX"                    # AWS Access Key
aws_secret_access_key: "XXXXXXXXXXXXX"                  # AWS Secret Key
aws_default_key_name: "paasta-key"                      # AWS Key Name
aws_default_security_groups: ["paasta-security"]        # AWS Security-Group
aws_region: "ap-northeast-2"                            # AWS Region

## MULTI-CPI VARIABLE :: OpenStack
openstack_auth_url: "http://XX.XXX.XX.XX:XXXX/v3/"      # OpenStack Keystone URL
openstack_username: "XXXXXX"                            # OpenStack User Name
openstack_password: "XXXXXX"                            # OpenStack User Password
openstack_domain: "XXXXXX"                              # OpenStack Domain Name
openstack_project: "PaaS-TA"                            # OpenStack Project
openstack_region: "RegionOne"                           # OpenStack Region
openstack_default_key_name: "paasta-key"                # OpenStack Key Name
openstack_default_security_groups: ["paasta-security"]  # OpenStack Security Group

... ((생략)) ...
```

> vi multi-cpi/cpi-config.yml (사용할 정보를 주석 해제한다.)
```
#### DIFFERENT IAAS CPI

cpis:
- name: aws-cpi
  type: aws
  properties:
    access_key_id: ((aws_access_key_id))
    secret_access_key: ((aws_secret_access_key))
    default_key_name: ((aws_default_key_name))
    default_security_groups: ((aws_default_security_groups))
    region: ((aws_region))

- name: openstack-cpi
  type: openstack
  properties:
    auth_url: ((openstack_auth_url))
    username: ((openstack_username))
    api_key: ((openstack_password))
    domain: ((openstack_domain))
    project: ((openstack_project))
    region: ((openstack_region))
    default_key_name: ((openstack_default_key_name))
    default_security_groups: ((openstack_default_security_groups))
    human_readable_vm_names: true

... ((생략)) ...
```

- CPI Config 적용
```
$ bosh update-cpi-config multi-cpi/cpi-config.yml -l multi-cpi/cpi-vars.yml
```

<br>

### <div id='2.4.3'/>2.4.3. Cloud Config 설정
Cloud Config에 대한 추가 설정을 진행한다.  
Same IaaS AZ의 경우 paasta-deployment 폴더의 cloud-config 파일을 이용하며, Different IaaS AZ의 경우 bosh/multi-cpi 폴더의 cloud-config 파일을 이용한다.  

#### <div id='2.4.3.1'/>2.4.3.1. Same IaaS AZ의 경우

```diff
cloud-config 의 azs 에서 각 인프라의 cpi-name 을 지정

 azs:
 - cloud_properties:
     availability_zone: ap-northeast-1a
   name: z1
+  cpi: aws-cpi
 - cloud_properties:
     availability_zone: ap-northeast-1a
   name: z2
+  cpi: aws-cpi
 - cloud_properties:
     availability_zone: ap-northeast-2a
   name: z3
+  cpi: aws-cpi-second
 - cloud_properties:
     availability_zone: ap-northeast-2a
   name: z4
+  cpi: aws-cpi-second
 ...
 ...
```

- Cloud Config 적용
```
$ bosh update-cloud-config ~/workspace/cloud-config/{iaas}-cloud-config.yml 
```

#### <div id='2.4.3.2'/>2.4.3.2. Different IaaS AZ의 경우

|파일명|설명|
|------|---|
| cloud-config-aws-openstack.yml | AWS, OpenStack cloud config file |
| cloud-config-openstack-vsphere.yml	| OpenStack, vSphere cloud config file |
| cloud-config-vsphere-aws.yml	 | vSphere, AWS cloud config file |

```
# AWS - OpenStack or OpenStack - AWS를 사용하는 경우
$ bosh update-cloud-config ~/workspace/bosh/multi-cpi/cloud-config-aws-openstack.yml

# OpenStack - vSphere or vSphere - OpenStack를 사용하는 경우
$ bosh update-cloud-config ~/workspace/bosh/multi-cpi/cloud-config-openstack-vsphere.yml

# vSphere - AWS or AWS - vSphere를 사용하는 경우
$ bosh update-cloud-config ~/workspace/bosh/multi-cpi/cloud-config-vsphere-aws.yml
```

- Cloud Config 적용
```
$ bosh update-cloud-config ~/workspace/bosh/multi-cpi/cloud-config-{iaas}-{iaas}.yml 
```

<br>

### <div id='2.4.4'/>2.4.4. Stemcell 업로드
설치된 BOSH에 로그인 후 사용하는 IaaS의 Stemcell 업로드를 진행한다.   
(e.g. AWS와 OpenStack의 두개의 환경을 사용 할 경우 해당 명령어를 두개 다 실행한다.)  
```
# paasta-deployment v5.6.5와 동일한 Stemcell인 ubuntu-bionic 1.34를 사용한다.
# AWS 스템셀의 경우 light Stemcell을 이용한다

# AWS
$ bosh upload-stemcell https://storage.googleapis.com/bosh-aws-light-stemcells/1.34/light-bosh-stemcell-1.34-aws-xen-hvm-ubuntu-bionic-go_agent.tgz --fix

# OpenStack
$ bosh upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/1.34/bosh-stemcell-1.34-openstack-kvm-ubuntu-bionic-go_agent.tgz --fix

# vSphere
$ bosh upload-stemcell https://storage.googleapis.com/bosh-core-stemcells/1.34/bosh-stemcell-1.34-vsphere-esxi-ubuntu-bionic-go_agent.tgz --fix
```

<br>


### <div id='2.4.5'/>2.4.5. Multi CPI를 이용한 AP 설치 테스트
Multi CPI 설정을 완료한 뒤, PaaS-TA AP를 설치하여 상호간 통신이 원활하게 진행되는지 테스트를 진행한다.  
PaaS-TA AP에 필요한 runtime-config 설정이나 변수 설정에 관한 설명은 PaaS-TA AP 가이드를 참조한다.  

본 가이드에서는 여러 케이스중 AWS - OpenStack 기준으로 Diego-cell을 OpenStack에, 나머지 VM을 AWS에 설치하여 진행하였다.  
Diego-cell뿐 아니라 다른 VM도 분산 배포가 가능하고 Diego-cell을 각각 다른 IaaS에 분산하여 배포도 가능하니 해당되는 설정에 맞게 배포 방식을 변경하여 설치를 진행한다.  

- PaaS-TA AP 설치 폴더 이동
```
$ cd ~/workspace/paasta-deployment/paasta
```

- diego-cell zone 변경
> $ vi vars.yml
```
... ((생략)) ...

# DIEGO-CELL
diego_cell_azs: ["z4", "z5"]		# Diego-Cell 가용 존
diego_cell_instances: 3			# Diego-Cell 인스턴스 수

... ((생략)) ...
```

- PaaS-TA AP 설치
```
$ source deploy-aws.sh
```


PaaS-TA AP 설치 완료 후 Test APP을 Push하여 App이 정상작동하는지 확인한다.




### [Index](https://github.com/okpc579/paasta-guide-new/blob/main/README.md) > [AP Install](../README.md) > PaaS-TA Multi CPI
