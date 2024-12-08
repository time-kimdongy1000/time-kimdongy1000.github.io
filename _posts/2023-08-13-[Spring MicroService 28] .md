---
title: Spring MicroService 28 Spring MicroService 분산 추적 3
author: kimdongy1000
date: 2023-08-13 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  zipkin]
math: true
mermaid: true
---

## 집킨
집킨은 마이크로서비스에서 요청의 흐름을 분석하고 성능을 모니터링 하는데 사용됩니다 즉 우리가 앞에서 사용한 키바나는 로그를 바탕으로 각 어플리케이션간의 분산 로그를 추적하는 반면
집킨은 요청의 흐름으로 데이터를 시각화 하고 어플리케인션간의 성능을 모니터링 합니다 단 집킨과 키바나를 1:1 비율로 비교할 수 없습니다 서로 목적이 다르기 때문입니다 

## 집킨의 아키텍쳐

![1](https://github.com/user-attachments/assets/42b63eac-d731-4e9e-9417-c2c2db7ee5d6)

집킨은 어플리케이션과 바로 연결됩니다 앞에서 키바에서는 어플리케이션이 로그스태시랑 연결이 되었고 로그 스태시가 엘라스틱서치에 데이터를 전송 그리고 그 데이터를 키바나에서 쿼리하는 아키텍쳐였다면 집킨은 바로 연결해서 성능을 시각화 하고 그 후 해당 데이터를 엘라스틱서치로 영구적으로 보관하고 다시 쿼리할 수 있는 것입니다 

## 전체소스
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project-zipkin3?ref_type=heads>

## 의존성추가 
우리는 gateway , empClient , savingMoney 에 의존성을 추가하겠습니다 

```

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
    <version>2.2.3.RELEASE</version>
</dependency>

```

기존의 의존성을 제거하고 (슬루스 , 로그스태시) 슬루스 집킨을 의존성으로 넣어줍니다 

## 설정정보 추가 
각 어플리케이션이 집킨하고 연결이 되어어야 합니다 그래서 config-server 에 각 어플리케이션의 설정정보를 추가합니다 (gateway , empClient , savingMoney )

```
spring:
  zipkin:
    base-url: http://zipkin:9411
    enabled: true
  sleuth:
    sampler:
      probability: 1.0

```
3곳의 설정정보에 다음과 같이 설정정보를 넣어줍니다 이때 `sleuth.sampler.probability` 기본값은 10% 즉 10%만 전송하는것을 1.0 으로 변경하면 100% 슬루스 로그를 집킨으로 전송합니다  

## API 호출후 집킨화면

![1](https://github.com/user-attachments/assets/0fcc71db-5c57-4e21-a89f-fd670f29250a)

API 를 호출후 우측 상단에 TraceId 를 적어주면된다 그러면 해당 추적 ID 가 어디서 부터 시작해서 어떤 모듈들을 거치고 그때 상태 및 걸린시간 
지금은 총 3.6초가 걸린 api 입니다 

우측 아래를 보면 시간초별로 어떤 api 가 행해졌는지 알 수 있습니다 

이때 시간흐름을 비교해서 서버를 성능적인 면을 확실히 모니터링 할 수 있습니다