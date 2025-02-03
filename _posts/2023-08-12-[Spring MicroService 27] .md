---
title: Spring MicroService 27 Spring MicroService 분산 추적 2
author: kimdongy1000
date: 2023-08-12 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  kibana]
math: true
mermaid: true
---

지난시간에는 스프링 클라우드 슬루스를 활용해서 우리가 특별한 로직이 없어도 프레임워크가 알아서 판단해서 추적ID , 스팬ID 를 넣는것을 보았습니다 지난시간에는 단순히 로그를 보여주는것으로 그쳤는데 이번시간에는 해당 로그를 중앙 저장소에 쌓고 (엘라스틱서치) 시각화(키바나) ELK 에 대해서 알아보겠습니다 로그스태시 , 엘라스틱서치 , 키바나는 지식이 많이 없으므로 최소한의 설정으로만 진행을 하도록 하겠습니다 

## 로그스태시 , 엘라스틱 서치 , 키바나 Docker 세팅
```
elasticsearch:    
    image: docker.elastic.co/elasticsearch/elasticsearch:8.7.0
    ports:
      - 9300:9300
      - 9200:9200
    environment:
      - node.name=elasticsearch
      - discovery.type=single-node
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false
      - xpack.security.transport.ssl.enabled=false
    volumes:
      - esdata1:/usr/share/elasticsearch/data      
    container_name: elasticsearch      

  logstash:
    image: docker.elastic.co/logstash/logstash:8.7.0
    ports:
      - "5044:5044"  
    volumes:
      - ./config:/etc/logstash/conf.d
    command: logstash -f /etc/logstash/conf.d/logstash.conf  
    container_name: logstash    

  kibana:
    image: docker.elastic.co/kibana/kibana:8.7.0
    ports:
      - 5601:5601    
    environment:
      ELASTICSEARCH_URL: "http://elasticsearch:9300"
    container_name: kibana      
```

어플리케이션은 슬루스 프레임워크로 인해서 발생한 로그들을 가져와서 로그스태시가 엘라스틱서치로 전송을 합니다 
엘라스틱서치는 해당 로그를 저장하고 
키바나는 엘라스틱 서치의 중앙저장소에 연결해서 로그들을 분석하여 시각화 하는 역활을 하게 됩니다 

## logstash conf.d
```
input {
  tcp {
    port => 5044
    codec => json_lines
  }
}

output {
  elasticsearch {
    hosts => "elasticsearch:9200"
  }
}

```
로그 스태시는 해당 conf 파일을 같이 매핑을 해주어야 하는데 이떄 input 는 어플리케이션단에서 보내는 로그들이 모이는 설정을 하는 곳이고 output 는 로그스태시에 input 된 로그들을 어디로 내보낼것인지에 대한 설정입니다 우리는 엘라스틱서치로 해당 주소를 적어두도록 하겠습니다 


## 로그가 전송되는 과정 

![1](https://github.com/user-attachments/assets/9d8cffe8-61df-49f2-b1a6-eadb8eecb08e)


로그가 전송되어서 운영 담당자가 보는 화면까지는 위의 그림처럼 향하게 된다 

Application 에서 로그가 발생이 되면 
로그스태시를 이용해서 엘라스틱 서치로 전송 및 저장이 된다 
그럼 운영 담당자는 키바나 쿼리를 통해서 해당 쿼리를 검색 하여 운영에 필요한 정보를 모우고 문제점을 진단하여 오류를 판단한다 


## 전체소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project-zipkin2?ref_type=heads>

## 키바나 접속 
사실 우리가 볼 수 있는 화면은 키바나 밖에 없다 물론 로그스태시나 , 엘라스틱서치로 들어가면 특별한 화면이 나오기는 하지만 크게 의미는 없을것이다 만약 설정을 올바르게 했다면 키바나가 바로 엘라스틱서치를 찾아서 연결이 된 모습을 볼 수 있을것이다 그렇지 않으면 키바나가 엘라스틱서치 연결을 요구하는데 이렇게 되면 dokcer 연결을 다시 한번 살펴보아야 한다 

```

kibana:
    image: docker.elastic.co/kibana/kibana:8.7.0
    ports:
      - 5601:5601    
    environment:
      ELASTICSEARCH_URL: "http://elasticsearch:9300"
    container_name: kibana      

```
이떄 환경변수 주소로 ELASTICSEARCH_URL 가 있는데 이 주소를 한번더 확인을 하면된다 

![1](https://github.com/user-attachments/assets/3c8dde8c-676a-4e90-b951-f34a52cb1ab5)

아마 첫화면 이런 화면이 보일것이다 

![1](https://github.com/user-attachments/assets/43f44a02-5f6d-448d-a7f1-f5cc4dd95151)

그럼 좌측에 삼지창 버튼을 누르면 Discover 을 클릭해서 들어오면 된다 이곳에서 Create Data View 를 클릭하자 

![1](https://github.com/user-attachments/assets/ce47b270-2850-40e0-b615-df0a4f03c3c9)

그럼 이곳에서 어떤 데이터를 볼 것인지 정할 수 있는데 우리는 일단 이곳에서 다음과 같이 입려하고 들어가자 (**) 는 모든 데이터를 다 보겠다는 뜻이다 

## 사원생성 실행시 로그 발생 
```

2024-12-07 20:43:12.053  INFO [miniProject-SavingMoney,25e4f5a12d866ace,c394bfefc8b220a2] 15844 --- [container-0-C-1] c.e.m.MiniProjectSavingMoneyApplication  : Received Message To EmpClient : CREATE , 0f118cd1-023b-42be-a326-775e02e94610
2024-12-07 20:43:12.053  INFO [miniProject-SavingMoney,25e4f5a12d866ace,c394bfefc8b220a2] 15844 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : ===============================================
2024-12-07 20:43:12.053  INFO [miniProject-SavingMoney,25e4f5a12d866ace,c394bfefc8b220a2] 15844 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : saveEmpRedisData : 
2024-12-07 20:43:12.053  INFO [miniProject-SavingMoney,25e4f5a12d866ace,c394bfefc8b220a2] 15844 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : ===============================================

```

사원을 생성한 로그 데이터다 이 데이터가 로그스태시로 전송이 되었다 우리는 그것을 검색을 해보자

![1](https://github.com/user-attachments/assets/dde50377-569d-4361-86b0-32f1e6d92c96)

키바나에서 검색을 하게 되는데 이것을 키바나 쿼리라고 한다 traceId:"25e4f5a12d866ace" 로 검색을 하게 되면 traceId 가 25e4f5a12d866ace 로그를 잡아서 쿼리하게 됩니다 
데이터는 총 4개의 Row 데이터가 나오게 되는데 이 데이터를 하나씩 살펴보겠습니다 제일 아래쪽에 있는 데이터가 시간상 제일 오래된 데이터입니다 


![1](https://github.com/user-attachments/assets/30d1dbf5-1fd6-4cb3-a9f9-3761bb23ef54)

이렇게 해당 로그의 우측에 파란색 버튼을 누르면 해당 JSON 데이터가 보이는데 우리는 이를 좀 분석을 해보도록 하겠습니다 


```
{
  "@timestamp": [
    "2024-12-07T11:43:01.728Z"
  ],
  "@version": [
    "1"
  ],
  "application_name": [
    "miniProject-EmpClient"
  ],
  "application.version": [
    "1.0"
  ],
  "data_stream.dataset": [
    "generic"
  ],
  "data_stream.namespace": [
    "default"
  ],
  "data_stream.type": [
    "logs"
  ],
  "level": [
    "INFO"
  ],
  "logger_name": [
    "com.example.miniprojectempclient.event.MessageSourceBean"
  ],
  "message": [
    "Seding message CREATE for empClient emp_id 0f118cd1-023b-42be-a326-775e02e94610"
  ],
  "spanId": [
    "25e4f5a12d866ace"
  ],
  "tags": [
    "manningPublications"
  ],
  "thread_name": [
    "http-nio-9100-exec-1"
  ],
  "trace.span_id": [
    "25e4f5a12d866ace"
  ],
  "trace.trace_id": [
    "25e4f5a12d866ace"
  ],
  "traceId": [
    "25e4f5a12d866ace"
  ],
  "_id": "DBTuoJMBEmOTo2qtJHnI",
  "_index": ".ds-logs-generic-default-2024.12.07-000001",
  "_score": null
}
```
아마 앞에 달려있는 tag 쿼리들만 보더라도 대강 어떤 역활을 하는지 알 수 있습니다 로그가 발생된 시간 , 메세지 , trace_id 가 있는 것을 볼 수 있습니다 이를 통해서 
운영 담당자는 해당 로그를 분석해서 볼 수 있습니다 그럼 이런 INFO 평범한 데이터 말고 주로 운영자들이 보게 될 데이터는 ERROR 이나 지연시간이 발생하는 것들입니다 

지연 시간이 발생되는 것에서는 구현이 조금 어려우니 에러 발생에 관련해서 키바나 검색시 어떻게 되는지 알아보겠습니다


## SavingMoney 에 에러추가 
```

public void saveEmpRedisData(EmpDao empDao)
{
    try{

        Random rnd2 = new Random();
        int rnd_number = rnd2.nextInt(10);
        if(rnd_number %  2 == 0){
            throw new RuntimeException("Error !!!!!!!!");
        }
    }
}

```
우리는 그럼 새로운 사원이 만들어질때 일정 비율로 실패를 만들겠습니다 그럼 키바나에서는 어떻게 보이는지 보겠습니다 

```

024-12-07 21:00:07.499  INFO [miniProject-SavingMoney,bd25f10ab9c8e426,6a55f6fad9158283] 2132 --- [container-0-C-1] c.e.m.MiniProjectSavingMoneyApplication  : Received Message To EmpClient : CREATE , fb0ac89e-e47a-4a71-be4b-2f222dc9c608
2024-12-07 21:00:07.500 ERROR [miniProject-SavingMoney,bd25f10ab9c8e426,6a55f6fad9158283] 2132 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : Error !!!!!!!!

```

같은 추적ID 에서 하나는 INFO 로그가 발생되었고 다른 하나는 ERROR 로그가 발생되었습니다 이때 마찬가지로 키바나 쿼리해보겠습니다 

## 키바나 쿼리
```

level:"ERROR" 

```
이렇게 입력을 하면 이제 에러가 발생된 로그들만 보이게 됩니다 

```
{
  "@timestamp": [
    "2024-12-07T12:00:16.502Z"
  ],
  "@version": [
    "1"
  ],
  "application_name": [
    "miniProject-SavingMoney"
  ],
  "data_stream.dataset": [
    "generic"
  ],
  "data_stream.namespace": [
    "default"
  ],
  "data_stream.type": [
    "logs"
  ],
  "level": [
    "ERROR"
  ],
  "level_value": [
    40000
  ],
  "logger_name": [
    "com.example.miniprojectsavingmoney.config.common.EmpClientRedisTemplate"
  ],
  "message": [
    "Error !!!!!!!!"
  ],
  "spanId": [
    "5cb7e815c25ec1ef"
  ],
  "tags": [
    "manningPublications"
  ],
  "thread_name": [
    "KafkaConsumerDestination{consumerDestinationName='empClient-topic', partitions=1, dlqName='null'}.container-0-C-1"
  ],
  "traceId": [
    "5e515a9c1c825cf0"
  ],
  "_id": "lxT9oJMBEmOTo2qt7nrd",
  "_index": ".ds-logs-generic-default-2024.12.07-000001",
  "_score": null
}
```
그중 하나의 에러를 가지고 와서 메세지 및 추적 아이디로 다시 분석을 하게 됩니다 어디에서 어떻게 문제가 발생되었는지 말이죠 

우리는 여기까지 해서 로그를 보내서 저장하고 시각화 하는 ELK 에 대해서 알아보았습니다 물론 프로그램 특정상 어떤 데이터를 시각화 할려면 로그 보내는것 부터 해서 키바나 까지 세팅을 해주어야만 진정으로 운영에서 사용할 수 있을만한 프로그램이 됩니다 저는 단순 기본세팅으로만 진행을 했기에 이 부분은 부족하여 다음 엘라스틱서치 , 키바나를 좀더 공부해서 
그럴듯한 백오피스 환경을 구성을 해보도록 하겠습니다 


