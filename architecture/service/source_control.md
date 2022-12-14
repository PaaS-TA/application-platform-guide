### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Source Control Service

## 목적
본 문서는 Application Platform (AP) - Source Control Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
Source Control Service는 사용자의 소스코드와 다양한 파일들을 안전하게 저장할 수 있는 저장소를 제공한다.  
기본적으로 커밋 히스토리, 브랜치 목록 등을 제공하고 다양한 정보를 웹 상으로 확인할 수 있는 UI를 제공한다.

![Source Control Service Architecture](image/source_control_architecture.PNG)

<br>

| 구분  | 스펙 |
|-------|-----|
| scm-server | 1vCPU / 2GB RAM / 30GB 추가 디스크 |
| mariadb | 1vCPU / 2GB RAM / 2GB 추가 디스크 |
| haproxy | 1vCPU / 2GB RAM / 2GB 추가 디스크 |
| sourcecontrol-webui | 1vCPU / 2GB RAM / 2GB 추가 디스크 |
| sourcecontrol-api | 1vCPU / 2GB RAM / 2GB 추가 디스크 |
| sourcecontrol-broker | 1vCPU / 2GB RAM / 2GB 추가 디스크 |


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Source Control Service
