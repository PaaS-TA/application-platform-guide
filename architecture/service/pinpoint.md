### [Index](https://github.com/okpc579/paasta-guide-new/blob/main/README.md) > [AP Architecture](../README.md) > Pinpoint APM Service

## 목적
본 문서는 Application Platform (AP) - Pinpoint APM Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도


![Pinpoint APM Service Architecture](image/pinpoint_architecture.png)

<br>

| 구분  | 스펙 |
|-------|------|
| collector | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| h_master | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| pinpoint_web | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| broker | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| haproxy_webui | 2vCPU / 8GB RAM / 30GB 추가 디스크 |



### [Index](https://github.com/okpc579/paasta-guide-new/blob/main/README.md) > [AP Architecture](../README.md) > Pinpoint APM Service
