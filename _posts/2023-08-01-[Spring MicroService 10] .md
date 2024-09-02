---
title: Spring MicroService 10 Spring MicroService Cloud-Config 활용법
author: kimdongy1000
date: 2023-08-01 15:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

우리는 지난시간에 GITHUB 연동을 통해서 config 서버 연동결을 해보았다 그럼 도대체 이것을 왜 알아야 하느냐 이다 결국 마이크로서비스의 아키텍텨를 잠깐 보여주겠습니다

![1](https://github.com/user-attachments/assets/8f6311c3-7f75-48cd-97e5-9d966051d074)

우리가 앞으로 배울 아키텍쳐이다 

클라이언트가 요청을 넣으면 요청은 각각의 서비스로 바로 전달되는것이 아니라 중간에 이 서비를 찾고 분배를 해주는 분배기가 적절한 서비스를 찾아서 요청을 중재를 하게 됩니다 

즉 사용자는 개별의 서비스 서버를 볼 수 없고 제일 앞에 노출되는 서비스 찾기 서버만 노출이 되고 이를 통해서 원하는 것을 진행을 하게 됩니다 

그렇기에 config 서버가 각각의 서버를 일사분란하게 통제를 해야 합니다 각각의 서버에서 직접 port 및 설정에 관련한 정보를 기입해도 문제가 되는 않지만 관리측면에서 어려워집니다 

그래서 config - server 에 각 모듈별 서비스를 저장하고 각 모듈의 모든 설정들은 한 방향으로만 바라보게 설정이 되어 있습니다 이게 간단한 아키텍쳐입니다 


그래서 오늘 우리가 할 것은 config - server 에서 모든 설정을 바라보게 하는 방법과 이를 도커 컨테이너로 만드는거 까지 진행을 하겠습니다 



총프로젝트는 3개를 만들것입니다 
config-server 
service1-module
service2-module

이때 maven 은 공통으로 한번만 기입하도록 하겠습니다

## maven

```

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.1</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
    <java.version>11</java.version>
    <spring-cloud.version>2021.0.8</spring-cloud.version>
</properties>


 <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bootstrap</artifactId>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

```

간단하게 이렇게 준비했습니다 이는 뒤에서 나올 서비스 1 하고 2 가 동일합니다 그렇기에 한번만 적도록 하겠습니다 

## config-server main

```
@SpringBootApplication
@EnableConfigServer
public class SpringCloudConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigApplication.class, args);
    }

}
```

## config-server bootstrap.yml

```
spring:
  application:
    name: config-server
  profiles:
    active:
      - native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config

server:
  port: 8087

```

## config-server /config/server1_module-dev.yml
```
server:
  port: 8081


```

## config-server /config/server2_module-dev.yml
```
server:
  port: 8082
```

먼저 config 서버를 세팅합니다 config 서버는 8087로 세팅을 하고 각각의 config 가 바라볼 파일들을 config 안에 명시를 한다고 하고 
`server1_module-dev.yml` , `server2_module-dev.yml` 이렇게 두겠습니다 이때는 다른 설정없이 일단 서버포트만 기동을 하겠습니다


## service1-module bootstrap.yml

```
spring:
  application:
    name: spring-cloud-service-module1

  config:
    import: "optional:configserver:http://localhost:8087/"

  cloud:
    config:
      name: service1_module
      profile: dev

```

## service2-module bootstrap.yml

```
spring:
  application:
    name: spring-cloud-service-module2

  config:
    import: "optional:configserver:http://localhost:8087/"

  cloud:
    config:
      name: service2_module
      profile: dev

```

이렇게 설정을 하고 나면 문제는 service1-module , service2-module2 는 각각 port 설정을 하지 않았기에 기본포트 8080으로 동작할려고 하지만 config 서버를 설정을 했으므로 
다양한 서버 설정을 바라보게 된다 그래서 실행을 하게 되면

```
Located property source: [BootstrapPropertySource {name='bootstrapProperties-configClient'}, BootstrapPropertySource {name='bootstrapProperties-classpath:/config/service1_module-dev.yml'}]

Tomcat initialized with port(s): 8081 (http)

Located property source: [BootstrapPropertySource {name='bootstrapProperties-configClient'}, BootstrapPropertySource {name='bootstrapProperties-classpath:/config/service2_module-dev.yml'}]

Tomcat initialized with port(s): 8082 (http)

```
이렇게 기본포트가 바뀌는것을 볼 수 있습니다 이렇게 하나의 서버에서 설정정보를 가지고 여러 어플케이션의 설정을 한번에 통제할 수 있습니다 여기 까지만 보면 지난시간 답습이라고 생각할 수 있습니다 우리는 이제 이것을 Docker 컨테이너를 만들어서 활용을 할것입니다 

## config-server Dockerfile

```
FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package -DskipTests

FROM openjdk:11-jdk-slim

RUN apt-get update && apt-get install -y curl

COPY --from=build /app/target/*.jar /config-server.jar

ENTRYPOINT ["nohup" , "java","-jar","/config-server.jar"]

```

지난시간처럼 만들어주는 대신 한가지 추가를 해야 합니다 config-server 이 올바르게 현재 동작이 되었는지 확인이 필요하기에  `RUN apt-get update && apt-get install -y curl`
스크립트를 추가해 줍니다 이는 다음 파일 DockerCompose 에서 다루겠습니다 

## service1-module Dockerfile
```

FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package -DskipTests

FROM openjdk:11-jdk-slim

COPY --from=build /app/target/*.jar /service1-module.jar

ENTRYPOINT ["nohup" , "java","-jar","/service1-module.jar"]

```

## service2-module Dockerfile
```

FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package -DskipTests

FROM openjdk:11-jdk-slim

COPY --from=build /app/target/*.jar /service2-module.jar

ENTRYPOINT ["nohup" , "java","-jar","/service2-module.jar"]

```

서비스 모듈의 Dockerfile 은 평이하게 만들어줍니다 

## Docker-compose 

```

services:
  configserver:
    build:
      context: ./spring-cloud-config
      dockerfile: Dockerfile
    ports:
      - "8087:8087"
    networks:
      - my-bridge-network
    healthcheck:
      test: curl --fail http://localhost:8087/actuator/health
      interval: 10s
      retries: 10
      start_period: 60s
      timeout: 10s
    container_name: configserver  
       
  service1-module:
    build:
      context: ./spring-cloud-service-module1
      dockerfile: Dockerfile
    ports:
        - "8081:8081"
    networks:
      - my-bridge-network
    depends_on:
      configserver:
        condition: service_healthy
        restart: true
    container_name: service1-module
     

  service2-module:
    build:
      context: ./spring-cloud-service-module2
      dockerfile: Dockerfile
    ports:
      - "8082:8082"
    networks:
      - my-bridge-network
    depends_on:
      configserver:
        condition: service_healthy
        restart: true
    container_name: service2-module
       
   
     
networks:
  my-bridge-network:
    driver: bridge

```

1. context 는 Dockerfile 이 어디에 있는지 명시를 해줄 수 있습니다 Dockerfile 이 위치한 디렉터리로 Docker 이미지를 만들게 됩니다 

2. networks 컨테이너 끼리 통신을 할때 사용해야 합니다 이에 대해서는 다음에 한반 다를 예정입니다 지금은 각 컨테이너끼리 통신을 해야 하는 상황일때 정의해서 사용합니다 

3. healthcheck 이 부분은 현재 해당 컨테이너가 올바르게 동작하는지에 대한 상태를 확인할 수 있습니다 

4. depends_on 이는 앞에 어떤 특정한 서비스가 실행이 되고 난 뒤에 실행되는 Docker 문법이지만 Docker 공식페이지에 가보면 이는 실행 순서를 보장하지 않는다고 되어 있습니다 
실제로 제가 여러번 테스트를 했을때 단순 depends_on 으로는 실행 순서를 보장할 수 없어서 찾은것이 아래에 있는 condition 

5. condition depends_on 과 같이 쓰이며 해당 컨테이너가 올바르게 동작을 하는지 확인을 할 때 사용합니다 즉 현재 docker-compose 문법으로는 
configserver 가 올바르게 동작을 해야 service1-module , service2-module 가 동작을 시작합니다 

## 문제점
이 스크립트를 실행을 하면 올바르게 실행이 되는것처럼 보이지만 실제로 service1-module service2-module 는 올바르게 실행이 안될것입니다 로그를 보겠습니다 

```
Fetching config from server at : http://localhost:8087/
Connect Timeout Exception on Url - http://localhost:8087/. Will be trying the next url if available

```
두개의 서비스 둘다 현재 이런 문제가 발생하게 됩니다 그 이유는 도커 컨테이너의 네트워크 세팅때문에 그렇게 됩니다 네트워크 이야기는 다음에 할 상황이 있을것이고 이애 대해서 우리는 수정을 해야 합니다 즉 이때 bridge 네트워크에서는 컨테이너끼리 통신할때 이름이 ip 주소가 됩니다 

즉 우리는 각각의 bootstrap.yml 에서 다음과 같이 수정을 하면됩니다 


## 서비스 bootstrap.yml 수정
```

import: "optional:configserver:${CONFIG_SERVER_URL:http://localhost:8087/}"

```
이렇게 수정을 하면 이제 Docker 에서 파라미터를 CONFIG_SERVER_URL 을 주게 되고 그것이 없으면 기본적인 서버주소 http://localhost:8087 를 사용하게 됩니다 

## docker-compose 수정

```

environment:
      CONFIG_SERVER_URL: http://configserver:8087/
    ports:
      - "8082:8082"

```
이렇게 추가를 하면 docker-compose 는 컨테이너 기동할때 CONFIG_SERVER_URL 알아서 집어넣게 됩니다 이렇게 하면 로컬에서 그리고 도커 환경에서 테스트 할 수 있는 
config 서버를 만들어보았습니다 

오늘 내용은 진짜 길었습니다 config 서버가 왜 필요한지 부터 시작해서 다양한 방법으로 docker-compose 작성까지 