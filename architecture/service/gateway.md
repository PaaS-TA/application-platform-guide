### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Gateway Service

## 목적
본 문서는 Application Platform (AP) - Gateway Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
Gateway Service는 오픈소스 API 관리 플랫폼으로 WSO2를 기반으로 구성한다.  
API 설계, 퍼블리싱, 액세스 관리 및 보안적용, API 통계, API rate limiting을 지원한다.  
멀티 테넌트(Multi-tenant) 기반의 shared 서비스가 아닌 사용자 조직 전용 api-gateway 서버(WSO2)를 제공한다.

![Gateway Service Architecture](image/gateway_architecture.png)

<br>

| 구분  | 스펙 |
|--------|-----|
| mariadb | 2vCPU / 4GB RAM / 10GB 추가 디스크 |
| service-broker | 2vCPU / 4GB RAM |
| api-gateway | 2vCPU / 4GB RAM / 20GB 추가 디스크 |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Gateway Service
