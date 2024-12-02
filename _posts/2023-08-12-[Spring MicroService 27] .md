---
title: Spring MicroService 26 Spring MicroService 분산 추적 2
author: kimdongy1000
date: 2023-08-12 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  sleuth]
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

어플리케이션은 슬루스 프레임워크로 인해서 발생한 로그들을 가져와서 엘라스틱서치로 전송을 합니다 
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
Application -> Logstash -> elasticsearch -> kibana 
(그림으로 표현)


## 