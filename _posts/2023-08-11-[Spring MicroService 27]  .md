---
title: Spring MicroService 26 Spring MicroService 분산 추적 2
author: kimdongy1000
date: 2023-08-11 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  sleuth]
math: true
mermaid: true
---

우리는 지난시간에 sleuth 를 이용해서 마이크로서비스에서 로깅을 추적해보았다 하지만 문제점이 있다 우리는 지금 엄청 소규모 작은 프로젝트에서 추적을 통해서 로깅을 확인했지만 독립적 모듈이 수십개에서 수백개에 해당하는 모든 독립모듈을 전부 이런식으로 추적할 수 없을것이다 그래서 sleuth 와 같이 쓰이는 오픈소스 ELK 스택에 대해서 알아볼것이ㅏ 

## ELK 
이는 어떤 오픈소스의 앞글자이다
Elasicsearch (엘라스틱서치) - 로그 데이터를 저장하고, 강력한 검색 및 분석 기능을 제공하는 분산 검색 엔진
Logstash (로그 스태시) - 로그 데이터를 수집하고, 필터링하여 Elasticsearch로 전송하는 데이터 처리 파이프라인
Kibana (키바나) - Elasticsearch의 데이터에 대한 대시보드와 시각화 도구를 제공 

## 슬루스와 ELK 의 결합 
마이크로서비스 프로그램들은 로그 데이터를 보내고자 로그 스태시와 통신을 합니다 로그 스태시는 로그를 분석해서 필요한 경우 엘라스틱서치에 해당 데이터를 전송합니다 엘라스틱서치에서 저장된 데이터는 가공되어서 클라이언트 프로그램 일종인 키바나를 통해서 검색 및 시각화 분석을 할 수 있습니다 



