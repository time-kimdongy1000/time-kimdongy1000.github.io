---
title: Spring MicroService 19 Spring MicroService Spring-Cloud-GateWay 2
author: kimdongy1000
date: 2023-08-04 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  GateWay]
math: true
mermaid: true
---

지난시간에는 스프링 게이트웨이를 구성하는 작업을 마쳤습니다 그럼 이제는 어떻게 사용자들이 게이트웨이 를 이용해서 해당 엔드포인트까지 도달 할 수 있는지 보겠습니다 

## 전체소스
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-gateway?ref_type=heads>

지난시간하고 변경된 부분은 일단 설정부분하고 , 클라이언트 부분을 각각 폴더로 분리했습니다 

## Server 구성파일
spring-cloud-eureka
spring-cloud-gateway
spring-config-server
docker-compose.yml


## client 구성파일
spring-cloud-enureKa-client
spring-cloud-enureKa-client2
spring-cloud-enureKa-client3
spring-cloud-enureKa-client4
docker-compose.yml

이렇게 변경한 이유는 좀더 유연하게 도커를 기동하고자 진행을 했습니다

## docker 전역 네트워크 구성 
`docker network create colud-network` 이렇게 전역으로 사용할 네트워크를 만들었습니다 이전에는 하나의 compose 파일 안에 네트워크가 구성이 되어 있었기 때문에 그 안에 있는 모든 인스턴스들이 동일한 네트워크 구성에 들어갈 수 있었는데 폴더와 컴포즈 파일이 분리됨에 따라 전역으로 사용할 네트워크가 필요해서 만들었습니다 

```

#networks:
#  eureka-network:
#    driver: bridge

networks:
  default:
    external:
      name:  colud-network

```
앞으로 사용할 네트워크 설정입니다 이렇게 해야 도커컴포즈로 인해서 각 인스턴스가 분리되더라도 동일한 네트워크 구성에 들어갈 수 있습니다

## 지난시간 
![1](https://github.com/user-attachments/assets/c316810d-9701-4d41-9a45-3a5b07a33e1d) 

이 그림을 잘 보면 이제 클라이언트는 오롯이 게이트웨이하고만 통신을 진행을 해야 합니다 절대로 다른 클라이언트와 직접적으로 통신을 할 수 없습니다 
그런환경을 만들기 위해서 제가 지난시간에 한 행동은 docker-compose 를 통해서 모든 port 를 지웠습니다 즉 외부에서는 절대로 직접 해당 클라이언트에 직접 도달 할 수 없게 만들었습니다 그럼 사용자는 어떻게 해당 모듈의 엔드포인트에 도달 할 수 있을까?


## 게이트웨이는 프록시 
게이트웨이는 프록시라고 알고 있으면됩니다 프록시 서버는 중계서버로 사용자와 모듈 가운데에서 중계해서 요청을 주고 받을 수 있게 도와줍니다 
<http://게이트웨이서버:유레카서버에등록된-클라이언트아이디/요청할엔드포인트>

이게 무슨말이냐

```
{
    "predicate": "Paths: [/spring-cloud-eureka-client1/**], match trailing slash: true",
    "metadata": {
        "management.port": "9000"
    },
    "route_id": "ReactiveCompositeDiscoveryClient_SPRING-CLOUD-EUREKA-CLIENT1",
    "filters": [
        "[[RewritePath /spring-cloud-eureka-client1/?(?<remaining>.*) = '/${remaining}'], order = 1]"
    ],
    "uri": "lb://SPRING-CLOUD-EUREKA-CLIENT1",
    "order": 0
},
```
지난시간에 <http://localhost:8072/actuator/gateway/routes> 게이트웨이에 등록된 라우터들을 보기 위해 엔드포인트 입력시 보이는 값들입니다 이때 각각 Path 가 보이시나요 이게 이제 유레카서버에 등록된 Path 이고 게이트웨이 서버는 이를 근거로 자신이 어디로 요청을 넣을지 결정을 합니다 

즉 사용자는 이렇게 요청할것입니다 `http://localhost:8072/spring-cloud-eureka-client1/create_emp` 이렇게 요청을 넣으면 도메인은 게이트웨이고 중간에 Path 는 게이트웨이 자신이 찾을 유레카서버에 등록된 클라이언트 그리고 마지막은 해당 클라이언트 안에 있는 엔드포인트입니다 

![1](https://github.com/user-attachments/assets/1ae34fa8-941f-45c8-9fc1-b4f7922aca70)

그러면 이제 하늘색으로 따라가보자 이제 이렇게 요청이 오게 됩니다 즉 모든 요청은 GateWay가 받게 되고 그 요청을 등록된 유레카서버에서 찾고 (사실 이미 등록되어 있기 때문에 다이렉트로 통신합니다 유레카서버를 거친다는 표현은 유레카서버에 등록이 되어 있어야 찾아갈 수 있다는 표현입니다) 그리고 등록된 클라이언트의 엔드포인트로 요청을 받을 수 있게 할 수 있습니다 




