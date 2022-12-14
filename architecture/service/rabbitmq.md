### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > RabbitMQ Service

## 목적
본 문서는 Application Platform (AP) - RabbitMQ Service의 Architecture를 제공한다.
<br><br>

## 시스템 구성도
RabbitMQ Service는 오픈소스 메세지 브로커 서버를 Shared 방식으로 제공한다.  
메세지를 많은 사용자에게 전달하거나, 요청에 대한 처리 시간이 길 때 해당 요청을 다른 API에게 위임하고 빠른 응답을 하고자 할 때 또는 MQ를 사용하여 애플리케이션 간 결합도를 낮추기 위해 사용할 수 있다.

![RabbitMQ Service Architecture](image/rabbitmq_architecture.png)

<br>

| 구분  | 스펙 |
|-------|-----|
| rmq-broker | 1vCPU / 2GB RAM  |
| rmq | 1vCPU / 2GB RAM  |
| haproxy | 1vCPU / 2GB RAM |



### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [AP Architecture](../README.md) > RabbitMQ Service
