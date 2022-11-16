### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > MongoDB Service

## 목적
본 문서는 Application Platform (AP) - MongoDB Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
MongoDB Service는 다중 Replica Set에 데이터를 분산 저장하는 구성이다.  
MongoDB는 샤딩(Sharding)을 사용하여 매우 큰 데이터 세트와 높은 처리량을 요하는 작업에 필요한 구성을 지원한다.  
MongoDB의 Replica Set은 동일한 데이터 세트를 유지하는 MongoDB 프로세스 그룹이며 중복성과 고가용성을 제공한다.  
Config server는 shard cluster에서 필요한 모든 메타 정보들을 저장한다.

![MongoDB Service Architecture](image/mongodb_architecture.png)

<br>

| 구분  | 스펙 |
|-------|----|
| mongodb-broker | 2vCPU / 4GB RAM |
| mongodb_shard | 2vCPU / 4GB RAM |
| mongodb_config | 2vCPU / 4GB RAM / 10GB 추가 디스크 |
| mongodb_master | 2vCPU / 4GB RAM / 10GB 추가 디스크 |
| mongodb_worker | 2vCPU / 4GB RAM / 10GB 추가 디스크 |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > MongoDB Service
