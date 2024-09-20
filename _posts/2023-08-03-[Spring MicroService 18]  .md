---
title: Spring MicroService 17 Spring MicroService Spring-Cloud-GateWay
author: kimdongy1000
date: 2023-08-03 12:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  GateWay]
math: true
mermaid: true
---

지난시간까지 클라이언트 회복성에 대해서 공부를 했습니다 이번시간은 제가 생각했을때의 핵심 부분인 스프링 클라우드 게이트웨이에 대해서 알아보겠습니다 
계속해서 반복하자면 독립적인 모듈들은 숨겨져 있으며 클라이언트는 실제 서비스하는 핵심서버를 알지 못한다 입니다 이 점을 기억하면 

## 진짜로 우리와 통신하는 서버 GateWay 
이제까지 우리는 숨겨진척 하는 독립적인 모듈과 통신을 했습니다 하지만 지금부터는 모든 서버를 숨길것입니다 그리고 클라이언트는 오롯이 GateWay 와 통신을 하면서 필요한 정볼르 주고 받을 수 있게 할것입니다 

## 리버스 프록시 
리버스 프록시는 자원에 도달하려는 클라이언트와 자원 사이에 위치한 중개서버 클라이언트는 어떤 서버와 통신하고 있는지 알지 못한다 리버스 프록시는 클라이언트 요청을 캡쳐한 후 클라이언트를 대신하여 원격자원을 호출한다 

## GateWay 전체 소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-gateway?ref_type=heads>


## 전체적인 그림
![1](https://github.com/user-attachments/assets/c316810d-9701-4d41-9a45-3a5b07a33e1d) 

이제 우리가 앞전에서 보았던 그림을 완성할 수 있습니다 이제 클라이언트는 Spring-GateWay 와 통신을 할것입니다 그리고 나머지 모든 서버들은 가려서 보이지 않게 만들것이고 내부통신으로만 통신을 하게끔 설정을 할것입니다 

## GatwWay 설정

```
server:
  port: 8072

spring:
  cloud:
    gateway:
      discovery.locator:
        enabled: true
        lowerCaseServiceId: true

eureka:
  instance:
    preferIpAddress : true
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone : "${EUREKA_SERVER_URL:http://eurekaserver:8070/eureka/}"
      #defaultZone : "${EUREKA_SERVER_URL:http://localhost:8070/eureka/}"


management:
  endpoints:
    web:
      exposure:
        include: '*'
    health:
      show-details: always
```
설정은 간단합니다 그리고 우리는 GateWay 의 포트만 노출을 할것이고 나머지 설정은 이전 포스터에서 진행을 했으니 나머지 설정은 생략하겠습니다 

## docker-compse

```
services:
  configserver:
    build:
      context: ./spring-config-server
      dockerfile: Dockerfile
    environment:
      EUREKA_SERVER_URL: http://eurekaserver/8070/eureka
    networks:
      - "eureka-network"      
    healthcheck:
      test: curl --fail http://localhost:8087/actuator/health
      interval: 10s
      retries: 10
      start_period: 60s
      timeout: 10s
    container_name: configserver  

  eurekaserver:
    build:
      context: ./spring-cloud-eureka
      dockerfile: Dockerfile
    environment:
      CONFIG_SERVER_URL : http://configserver:8087/
    networks:
      - "eureka-network"
    depends_on:
      configserver:
        condition: service_healthy
        restart: true
    container_name: eurekaserver
   
  eurekaclient1:
    build: 
      context: ./spring-cloud-enureKa-client
      dockerfile: Dockerfile
    environment:
      CONFIG_SERVER_URL: http://configserver:8087/
    networks:
      - "eureka-network"      
    depends_on:
      configserver:
        condition: service_healthy
        restart: true
    container_name: eurekaclient1      
  
  eurekaclient2:
    build: 
      context: ./spring-cloud-eureka-client2
      dockerfile: Dockerfile
    environment:
      CONFIG_SERVER_URL: http://configserver:8087/
    networks:
      - "eureka-network"      
    depends_on:
      configserver:
        condition: service_healthy
        restart: true
    container_name: eurekaclient2
  
  eurekaclient3:
    build: 
      context: ./spring-cloud-eureka-client3
      dockerfile: Dockerfile
    environment:
      CONFIG_SERVER_URL: http://configserver:8087/
    networks:
      - "eureka-network"      
    depends_on:
      configserver:
        condition: service_healthy
        restart: true
    container_name: eurekaclient3
  
  springgateway:
    build:
      context: ./spring-cloud-gateway
      dockerfile: Dockerfile
    environment:
      CONFIG_SERVER_URL: http://configserver:8087/      
    ports:
      - 8072:8072      
    networks:
      - "eureka-network"            
    depends_on:
      configserver:
        condition: service_healthy
        restart: true      
    container_name: springgateway
networks:
  eureka-network:
    driver: bridge

```
이제 앞으로 docker-compose 파일은 다음과 같이 gateway 를 제외한 모든 포트를 숨길것입니다 그래서 클라이언트가 직접 서비스에 접근하는것을 막고 반드시 리버스 프록시 서버인 gateway 를 통해서 서버를 호출하게끔 만들예정입니다 

그렇게 실행을 하게 되면 우리가 유일하게 접근이 가능한 게이트웨이 서버로 actual 을 날려보면 다음과 같이 나오게 됩니다 

```
GET /actuator/gateway/routes HTTP/1.1
Host: localhost:8072


[
    {
        "predicate": "Paths: [/spring-cloud-eureka-client2/**], match trailing slash: true",
        "route_id": "ReactiveCompositeDiscoveryClient_SPRING-CLOUD-EUREKA-CLIENT2",
        "filters": [
            "[[RewritePath /spring-cloud-eureka-client2/?(?<remaining>.*) = '/${remaining}'], order = 1]"
        ],
        "uri": "lb://SPRING-CLOUD-EUREKA-CLIENT2",
        "order": 0
    },
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
    {
        "predicate": "Paths: [/spring-cloud-gateway/**], match trailing slash: true",
        "metadata": {
            "management.port": "8072"
        },
        "route_id": "ReactiveCompositeDiscoveryClient_SPRING-CLOUD-GATEWAY",
        "filters": [
            "[[RewritePath /spring-cloud-gateway/?(?<remaining>.*) = '/${remaining}'], order = 1]"
        ],
        "uri": "lb://SPRING-CLOUD-GATEWAY",
        "order": 0
    }
]
```
자 그럼 이제 우리는 이 정보를 가지고 어떻게 gateway 가 필요한 api 를 매핑할 수 있는지는 다음 포스트에서 부터 배워보도록 하겠습니다 
