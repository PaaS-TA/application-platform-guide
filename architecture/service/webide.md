### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > WEB IDE Service

## 목적
본 문서는 Application Platform (AP) - WEB IDE Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
WEB-IDE 서비스는 eclipse che를 이용하여 웹 기반의 개발 가능한 IDE를 제공한다.  
멀티 테넌트(Multi-tenant) 기반의 shared 서비스가 아닌 사용자 전용 eclipse che 서버를 제공한다.  
eclipse che 서버가 기본적으로 제공하는 stack 을 기반으로 하고 있기에 workplace 구성에 있어 제한적이다.
 
![WEB IDE Service Architecture](image/webide_architecture.png)

<br>

| 구분  | 스펙 |
|-------|-----|
| eclipse-che | 2vCPU / 8GB RAM |
| mariadb | 1vCPU / 2GB RAM / 10GB 추가 디스크 |
| webide-broker | 2vCPU / 4GB RAM |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > WEB IDE Service
