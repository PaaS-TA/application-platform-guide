### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > MongoDB Service

## 목적
본 문서는 Application Platform (AP) - MongoDB Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
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
