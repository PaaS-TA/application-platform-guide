### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Redis Service

## 목적
본 문서는 Application Platform (AP) - Redis Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
Redis 서비스를 통해 인 메모리 캐시 저장소를 제공한다.  
멀티 테넌트(Multi-tenant) 기반의 shared 서비스가 아닌 사용자 on-demand Redis 서버를 제공한다.

![Redis Service Architecture](image/redis_architecture.png)

<br>

| 구분  | 스펙 |
|-------|-----|
| mariadb | 2vCPU / 4GB RAM / 2GB 추가 디스크 |
| paas-ta-on-demand-broker | 2vCPU / 4GB RAM |
| redis | 2vCPU / 4GB RAM / 1GB 추가 디스크 |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Redis Service
