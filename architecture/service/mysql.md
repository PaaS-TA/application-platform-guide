### [Index](https://github.com/okpc579/paasta-guide-new/blob/main/README.md) > [AP Architecture](../README.md) > MySQL Service

## 목적
본 문서는 Application Platform (AP) - MySQL Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
<br>

![MySQL Service Architecture](image/mysql_architecture.png)

<br>

| 구분  | 스펙 |
|-------|----|
| arbitrator | 1vCPU / 2GB RAM |
| mysql | 1vCPU / 2GB RAM / 8GB 추가 디스크 |
| mysql_broker | 1vCPU / 2GB RAM |
| proxy | 1vCPU / 2GB RAM |



### [Index](https://github.com/okpc579/paasta-guide-new/blob/main/README.md) > [AP Architecture](../README.md) > MySQL Service
