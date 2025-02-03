---
title: Spring MicroService 6 Spring MicroService Cloud-Config 1
author: kimdongy1000
date: 2023-08-01 12:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

## 모놀리스 구성정보 
아마 대부분의 개발자 모놀리스로 프로젝트를 구성할때 구성 정보는 대부분 하나의 프로젝트 안에서 properties 로 만들어지고 각각의 런타임 환경에 따라 local , dev , prod 이렇게 관리되고 있을것이다 이때 핵심은 하나의 어플리케이션 안에 구성정보가 함께 배포된다는 점이다 모놀리스 프로젝트 같은 경우는 하나의 어플리케이션과 , 구성정보가 1:1 매핑이기 때문에 이를 관리하기 쉽다 
하지만 클라우스 서비스는 수백개의 독립적인 모듈이 존재한다 이때 독립적인 모듈 안에 모놀리스 식으로 소스코드와 함께 구성정볼르 배포하는 일은 위험하기 짝이 없는 행동이다

## 마이크로서비스 구성정보
예를 들어 각각의 독립된 서비스들이 소스코드 안에 구성정보를 담아서 배포를 한다고 생각을 해보자 물론 할 수 있다 다만 문제점은 다음에서 부터 발생한다 예를 들어서 prod DB 서버가 문제가 생겨서 
서버 물리적인 주소를 바꾸었다고 생각을 해버보자 그럼 소스코드와 함깨 빌드되 배포된 구성정보들을 일일이 다시 꺼내서 수정을 해야 하는 위험성이 있다 
그래서 마이크로서비스 구성정보는 소스코드와 구성정보를 함께 빌드해서 배포하는것을 매우 위험하게 생각한다 그래서 다양한 플랫폼들이 이런 위험성에서 벗어나게 해주는데 그 중 우리는 
Spirng 에서 제공하는 Cloud - Config 서버에 대해서 알아볼것이다 

## Cloud - Config Server 아키텍쳐
이 방식은 구성정보를 개별 소스코드에 저장해서 빌드해서 배포하는 것이 아니라 구성정보만 담고 있는 독립적인 모듈을 배포하고 각 독립적인 모듈들은 런타임시 Cloud - Config Server 의 구성정보를 읽어와서 이를 주입받는 식으로 개발이 되어야 합니다 그럼 우리는 먼저 구성정보를 저장하는 중앙서버 부터 만들어보겠습니다

## maven 
```

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.1</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>project.amadeus</groupId>
	<artifactId>Project_Spring_Cloud-config</artifactId>
	<version>1.0.0</version>
	<name>Project_Spring_Cloud-config</name>
	<description>Proejct_Amadues</description>
	<url/>
	<licenses>
		<license/>
	</licenses>
	<developers>
		<developer/>
	</developers>
	<scm>
		<connection/>
		<developerConnection/>
		<tag/>
		<url/>
	</scm>
	<properties>
		<java.version>11</java.version>
		<spring-cloud.version>2021.0.8</spring-cloud.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
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

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>



```

복잡해 보이지만 간단하게 web , spring-cloud-config-server 를 추가를 해주면된다 이때 spring - cloud 는 spring boot stater 에 의해서 관리되는것이 아니기 때문에 사용하는 boot 버전이 
spring - cloud 버전과 호환이 맞아야 한다 그에 대한 내용은 

<https://spring.io/projects/spring-cloud#overview> 참고하면 됩니다 


## application.yml 
```
spring:
  application:
    name: config-server
  profiles:
    active: native

    cloud:
      config:
        server:
          native:
            search-locations: classpath:/config


server:
  port: 8087

```

yml 파일을 이와 같이 작성을 하겠습니다 그리고 핵심은 search-locations 우리는 이 search-locations 을 통해서 이 프로젝트가 중앙저장소로의 역활을 할때 구성정보 위치를 명시를 해주는 것입니다 
지금 classpath:/config 라면 이 프로젝트르 resource 아래에 있는 config 를 중앙저장소로 인식을 할것입니다 

## @EnableConfigServer

```

package project.amadues.source;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ProjectSpringCloudConfigApplication {

	public static void main(String[] args) {
		SpringApplication.run(ProjectSpringCloudConfigApplication.class, args);
	}

}


```
그리고 @EnableConfigServer 애너테이션을 통해서 이 프로젝트가 spring - config 서버로 활용된다는 것을 명시합니다 

## 구성정보 만들기
우리는 총 3개의 구성정보를 만들것입니다 

## config-service.properties
```

## default-properties

spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000

```

## config-service-dev.properties
```

## dev-properties

spring.datasource.url=jdbc:mysql://dev:3306/dev-database
spring.datasource.username=dev-user
spring.datasource.password=dev-password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

```

## config-service-prod.properties

```

## prod-properties

spring.datasource.url=jdbc:mysql://prod:3306/dev-database
spring.datasource.username=prod-user
spring.datasource.password=prod-password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

```

이렇게 총 3개의 접속정보를 만듭니다 이 파일들은 classpath:/config 아래에 저장이 됩니다 그럼 이를 실행을 하겠습니다 이때 실행은 docker 로 하겠습니다 

## Docker 파일 작성
```

FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package

FROM openjdk:11-jdk-slim

COPY --from=build /app/target/*.jar /config-server.jar

ENTRYPOINT ["java","-jar","/config-server.jar"]

```

## Docker-compose.yml 

```
version: '3.8'

services:
  app:
    build: .
    container_name: config-server
    ports:
      - "8087:8087"


```

우리는 이제 로컬로 실행하기 보다 Docker 와 친해지기 위해 계속 간단하더라도 도커 스크립트를 활용할 것입니다 

## post - man 
우리는 이제 요청을 하겠습니다 요청은 총 3개로 

## http://localhost:8087/config-service/default
```

{
    "name": "config-service",
    "profiles": [
        "default"
    ],
    "label": null,
    "version": null,
    "state": null,
    "propertySources": [
        {
            "name": "classpath:/config/config-service.properties",
            "source": {
                "spring.datasource.hikari.connection-timeout": "30000",
                "spring.datasource.hikari.minimum-idle": "10",
                "spring.datasource.hikari.maximum-pool-size": "10",
                "spring.datasource.hikari.idle-timeout": "600000",
                "spring.datasource.hikari.max-lifetime": "1800000"
            }
        }
    ]
}

```
지금 보면 요청을 하게 되면 이처럼 JSON 형태로 정보가 반환이 됩니다 이때는 기본 정보 즉 우리가 properties 에서 아무것도 적지 않은 기본 properties 가 반환이 되는 것입니다


## http://localhost:8087/config-service/dev

```

{
    "name": "config-service",
    "profiles": [
        "dev"
    ],
    "label": null,
    "version": null,
    "state": null,
    "propertySources": [
        {
            "name": "classpath:/config/config-service-dev.properties",
            "source": {
                "spring.datasource.url": "jdbc:mysql://dev:3306/dev-database",
                "spring.datasource.username": "dev-user",
                "spring.datasource.password": "dev-password",
                "spring.datasource.driver-class-name": "com.mysql.cj.jdbc.Driver"
            }
        },
        {
            "name": "classpath:/config/config-service.properties",
            "source": {
                "spring.datasource.hikari.connection-timeout": "30000",
                "spring.datasource.hikari.minimum-idle": "10",
                "spring.datasource.hikari.maximum-pool-size": "10",
                "spring.datasource.hikari.idle-timeout": "600000",
                "spring.datasource.hikari.max-lifetime": "1800000"
            }
        }
    ]
}

```

dev 는 좀더 내용이 늘어낫습니다 이는 기본정보 + dev.properties 정보가 포함되어 return 이 되는 것입니다 

## http://localhost:8087/config-service/prod

```

{
    "name": "config-service",
    "profiles": [
        "prod"
    ],
    "label": null,
    "version": null,
    "state": null,
    "propertySources": [
        {
            "name": "classpath:/config/config-service-prod.properties",
            "source": {
                "spring.datasource.url": "jdbc:mysql://prod:3306/dev-database",
                "spring.datasource.username": "prod-user",
                "spring.datasource.password": "prod-password",
                "spring.datasource.driver-class-name": "com.mysql.cj.jdbc.Driver"
            }
        },
        {
            "name": "classpath:/config/config-service.properties",
            "source": {
                "spring.datasource.hikari.connection-timeout": "30000",
                "spring.datasource.hikari.minimum-idle": "10",
                "spring.datasource.hikari.maximum-pool-size": "10",
                "spring.datasource.hikari.idle-timeout": "600000",
                "spring.datasource.hikari.max-lifetime": "1800000"
            }
        }
    ]
}

```

그리고 마지막 prod 도 prod.properties 를 반환을 해줍니다 


## 원리
@EnableConfigServer 를 명시하면 이제 spring 은 구성정보를 반환랄려고 내부에서 준비합니다 그리고 우리는 그 파일의 위치를 명시를 해주었습니다 `search-locations: classpath:/config`
이렇게 말이죠 그럼 이 프로젝트는 기동과 동시에 config 정보를 JSON 형태로 만들어 놓고 HTTP 형태의 요청이 들어오면 이를 JSON 형태로 return 을 해줍니다 

```

http://localhost:8087/config-service/default
http://localhost:8087/config-service/dev
http://localhost:8087/config-service/prod

```
이런 요청정보로 return 이 되는 이유는 @EnableConfigServer 서버가 자동으로 config 에 매핑된 이름정보를 가지고 해당 config 를 반환합니다 dev -> dev.properties 를 반환
prod -> prod.properties 반환 하게 됩니다 

## 정리 
클라우드 컨피그 서버는 구성정보를 중앙에서 저장을 하고 http 요청이 오면 이를 JSON 형태로 반환을 하게 됩니다 우리는 이제 다음장에서는 스프링 클라우드 컨피그 클라이언트를 구축해서 
중앙서버에 있는 정보를 각각 어떻게 읽는지에 대해서 서술을 할것입니다 


