---
title: Spring MicroService 2 Spring MicroService 와 Docker - Dockerfile
date: 2023-08-01 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService , Docker ]
math: true
mermaid: true
---

지난장에서는 Docker 와 관련한 전반적인 내용을 다루어보았습니다 얕게나마 개념정도 알고 진행을 했습니다 이번시간에는 직접 웹 어플리케이션을 만들고 그 어플리케이션을 도커 이미지화 해서 만들어서 실행하는 방법까지 진행을 해보도록 하겠습니다

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
</properties>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

버전은 그렇게 크게 신경쓰지 말도록 하고 web 에 대한 의존성을 걸기만 할 예정입니다 

## HelloDockerController

```
@RestController
public class HelloDockerController {
	
	@GetMapping("/")
	public String helloDocker() {
		
		return "helloDocker";
	}

}

```
간단한 핸들러를 만들겠습니다 

## 도커엔진 시작 

![docker_b](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f94dbf74-fc9e-4ca1-ad24-63618273bcbe)

설치한 도커엔진을 기동하면 이런 모양으로 나오게 됩니다 저는 이미지 몇개를 만들어본 터라 지금 이런 형태입니다 그럼 도커엔진에게 이미지 파일을 만들기 전에 해야 할 일은
Dockerfile 을 만드는 것입니다 

## Dockerfile 
도커 클라이언트가 이미지를 생성하고 준비하기 위해 호출하데 필요한 지시어와 명령어들이 포함된 단순한 테스트 파일입니다 이 파일은 이미지 생성 과정을 자동화 합니다 
그럼 일단 간단하게 도커 파일을 만들어보겠습니다

```
FROM openjdk:11-jdk-slim

ARG JAR_FILE=target/*.jar

COPY ${JAR_FILE} app.jar

ENTRYPOINT ["java","-jar","/app.jar"]

```

Dockerfile 이라는 이름으로 (확장자x) 프로젝트 root 바로 아래에 만들어둡니다 그리고 cmd 로 다음의 명령어를 실행을 하겠습니다 

## 도커 빌드 

![docker_c](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/90171aa3-294f-4689-88f7-3034a9a7530c)

인텔리j 기준 cmd 를 켜서 해당 프로젝트 root 에서 아래의 명령어를 실행시키거나 
cmd 기준 해당 프로젝트 root 아래까지 와서 아래의 명령어를 실행을 시키면 됩니다 

```

docker build -t proejct-spring-boot-docker:0.0.1V .

```

그럼 도커 호스트에 다음과 같은 이미지가 생성이 됩니다 

![docker_d](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0218d43a-660d-4d97-8a0f-46afdc056d0d)

이 이미지가 바로 도커 컨테이너에 격리된 실행가능한 독립적인 이미지가 됩니다 그럼 이 이미지를 실행을 해보겠습니다 보이는 GUI 에서 실행버튼을 눌러도 되지만 

cmd 를 켜서 명령어로 진행을 하겠습니다 

## 도커 이미지 run 

```

docker run -p 5000:8080 proejct-spring-boot-docker:0.0.1V

```

이렇게 명령어를 써줍니다 다른 명령어들도 한번 분석을 할것인데 이 명령어만 바로 진행을 하겠습니다 -p 옵션은 5000 번으로 들어오는 port 를 8080 으로 매핑을 해줘라 뜻입니다 
어디서 많이 보는 방식이죠 그렇습니다 도커를 실행할때 포트포워딩을 진행할 수 있습니다 그럼 5000 번으로 들어오는 요청들은 도커가 자동으로 8080으로 매핑을 시켜줄 것입니다 

## 웹사이트 요청

```
http://localhost:5000

```

우리는 이렇게 요청을 넣게 되면 도커가 알아서 8080에 매핑되어 있는 핸들러를 찾아서 보여주게 됩니다 우리는 그러면 크게 웹어플리케이션을 하나 만들었고 그것을 도커 클라이언트를 이용해서 이미지를 만들었고 호스트에 있는 도커 이미지를 실행하는데 실행할때 포트 번호를 포트포워딩해서 하는 작업을 진행을 했습니다 다음 장에서는 이번시간에 쓰인 이미지 만들때 쓰인 명령어들에 대해서 조금 알아보도록 하겠습니다 






