### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Lifecycle Service

## 목적
본 문서는 Application Platform (AP) - Lifecycle Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
Lifecycle Service는 오픈소스 TAIGA를 기반으로 구성한다.  
애플리케이션의 개념 구상에서 부터 수명 종료까지 애플리케이션의 라이프사이클 관리하는데 필요한 인력, 각종 문서, 이슈 등을 관리할 수 있는 UI를 제공한다.  
멀티 테넌트(Multi-tenant) 기반의 shared 서비스가 아닌 사용자 조직 전용 api-lifecycle 서버(taiga)를 제공한다.

![Lifecycle Service Architecture](image/lifecycle_architecture.png)

<br>

| 구분  | 스펙 |
|-------|----|
| mariadb | 2vCPU / 4GB RAM / 10GB 추가 디스크 |
| service-broker | 2vCPU / 4GB RAM |
| app-lifecycle | 2vCPU / 4GB RAM / 20GB 추가 디스크 |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Lifecycle Service
