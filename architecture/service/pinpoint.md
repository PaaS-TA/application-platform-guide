### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Pinpoint APM Service

## 목적
본 문서는 Application Platform (AP) - Pinpoint APM Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
Pinpoint APM Service는 사용자 어플리케이션의 구성 요소간에 트랜잭션 흐름을 추적하고 문제영역 및 잠재적인 병목 현상을 식별할 수 있도록 데이터를 수집하고 시각적으로 확인할 수 있도록 UI를 제공한다.


![Pinpoint APM Service Architecture](image/pinpoint_architecture.png)

<br>

| 구분  | 스펙 |
|-------|------|
| collector | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| h_master | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| pinpoint_web | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| broker | 2vCPU / 8GB RAM / 30GB 추가 디스크 |
| haproxy_webui | 2vCPU / 8GB RAM / 30GB 추가 디스크 |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > Pinpoint APM Service
