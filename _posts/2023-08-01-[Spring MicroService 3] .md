---
title: Spring MicroService 3 Spring MicroService 와 Docker - Dockerfile
author: kimdongy1000
date: 2023-08-01 10:30
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService , Docker ]
math: true
mermaid: true
---

지난상에서는 웹어플리케이션을 만들고 이를 도커 컨테이너를 통해서 실행하는 방법을 알아보았습니다 이번시간에는 앞에 쓰였던 Dockerfile 명령어에 대해서 알아보도록 하겠습니다 

## Dockerfile

```
FROM openjdk:11-jdk-slim

ARG JAR_FILE=target/*.jar

COPY ${JAR_FILE} app.jar

ENTRYPOINT ["java","-jar","/app.jar"]

```

우리가 도커이미지를 만들때 사용한 명령어 입니다 이게 무슨 역활이게 무슨뜻인지에 대해서 알아보도록 하겠습니다 



FROM : 
	빌드 프로세스를 시작하는 기본 이미지를 정의합니다 FROM 명령어는 도커 런타임에 사용할 도커 이미지를 지정합니다 모든 Dockerfile은 이 명령어로 시작해야 합니다.
	뒤에 있는 openjdk:11-jdk 사용할 기본이미지 사용할 JDK 의 버전을 명시해줍니다 이때 기본이미지는 우리가 만들 커스텀 이미지가 어떤 이미지를 바탕으로 만들어지냐는 것입니다 
	즉 우리가 만든 웹 어플리케이션은 java 기반으로 만들어졌습니다 그리고 pom.xml 에 명시한데로 11버전으로 맞춰 두었기 때문에 기본 이미지가 openjdk:11-jdk 입니다 
	그럼 뒤에 slim 은 무엇인지 즉 말그대로 슬림 날씬한 빌드를 하기 위한 최소한의 의존성만 가지고 있는 버전이라는 뜻입니다 Docker 의 경량화 정책에 맞게 가져오는 것입니다 


ARG :
	사용자가 docker build 명령을 사용하여 빌더에 전달할 수 있는 변수를 정의할 수 있습니다 지금 보면 2번째 줄에 ARG JAR_FILE=target/*.jar 이렇게 정의가 되어 있고 
	이 변수를 다음장 COPY 를 통해서 사용하고 있는 모습입니다 

COPY : 
	원본의 새 파일 디렉터리 또는 리모프 파일 URL 을 복사하고 지정된 대상 경로에 생성입니다 우리는 특별한 경로 없이 바로 도커 이미지 root 파일에 저장을 했습니다 
	C:\Users\[본인계정명]\AppData\Local\Docker\wsl\data 저장이 됩니다 

ENTRYPOINT : 
	실행 파일로 실행할 컨테이너를 구성한다 그럼 ENTRYPOINT ["java","-jar","/app.jar"] 명령은 컨테이너가 실행할때 java -jar /app.jar 이렇게 알아서 실행이 되는것입니다 

이 외에도 여러 명령어가 있는데 일단 이 정도로만 하고 다음 새로운 명령어가 나올때마다 정리를 하도록 하겠습니다 

## 도커 빌드 
```

docker build -t proejct-spring-boot-docker:0.0.1V .

```

우리는 앞부분에서 도커를 build 할때 이와 같은 명령어를 사용했습니다 이때 -t는 도커 이미지를 생성할때 태그를 달라는 뜻입니다 지금 proejct-spring-boot-docker: 이후가 태그 부분입니다 이때 해당 Srping 버전을 0.0.1V 로 하면 해당 컨테이너는 0.0.1V 가 만들어 집니다 예를 들어서 기능이 조금 변경이 되었었다고 생각을 하겠습니다 

핸들러 하나더 추가가 되었다고 생각을 해보겠습니다 

## 핸들러 추가 
```
@RestController
public class HelloDockerController {
	
	@GetMapping("/")
	public String helloDocker() {
		
		return "helloDocker";
	}
	@GetMapping("/v2")
	public String helloDockerV2() {

		return "helloDockerV2";
	}

}

```
지금처럼 v2 버전의 핸들러가 하나더 추가 되었다고 보고 이를 0.0.2 로 빌드를 해보겠습니다 

```

docker build -t proejct-spring-boot-docker:0.0.2V .

```
그럼 TAG 를 달때 이와 같은 모양으로 달고 도커호스트에는 0.0.1V 와 0.0.2V 버전이 같이 생기게 됩니다 이때 우리는 0.0.2V 를 실행하겠습니다 

그런데 새로 추가한 v2 라는 핸들러는 새롭게 만들어진 이미지에서는 호출되지 않습니다 그 이유는 무엇이냐면 소스코드가 바뀌었을때는 새롭게 maven 패키징 작업을 해주어야 합니다 


```
FROM openjdk:11-jdk-slim

ARG JAR_FILE=target/*.jar

COPY ${JAR_FILE} app.jar

ENTRYPOINT ["java","-jar","/app.jar"]

```
이 작업은 지금 현재 만들어져 있는 jar 파일을 도커 이미지화 하기 떄문에 최신화된 소스는 반영이 되지 않습니다 그럼 새롭게 핸들러가 수정될때 마다 다시 package 를 하는 것을 Dockerfile 에 추가를 하겠습니다 

## 수정 

```
FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package

FROM openjdk:11-jdk-slim

COPY --from=build /app/target/*.jar /app.jar

ENTRYPOINT ["java","-jar","/app.jar"]

```

아마 이것을 build 하게 되면 이전 보다는 시간이 많이 흐르게 될것입니다 일단 기본적으로 없는 것들 예를 들어서 maven 같은 경우는 중앙 도커 허브에서 다시 내려받는 과정을 거쳐야 합니다 그리고 docker 가 자체적으로 maven package 하는 과정을 거치게 됩니다 그런데 이 중앙 허브를 통해서 다운로드 하는것은 최초 1회입니다 그것을 저의 로컬에 캐싱을 해두고 그것을 사용하게 됩니다 그러면 이제 이 Dockerfile 는 변경된 소스코드가 있을때마다 새롭게 package 를 하고 해당 내용을 이미지화 합니다 

## 태그의 유무 
태그 이야기 하다가 여기 까지 왔습니다 태그의 의미가 뭘까? 그럼 잠시 지난시간으로 돌아가서 Docker 장점중 하나인 애플리케이션을 빠르게 빌드하고 배포할 수 있습니다.
이 빠르게 배포는 오류가 발생한 버전에 대해서 빠른 복구가 가능합니다 우리가 태그한 버전은 그 때 당시의 소스코드가 그대로 있습니다 

즉 0.0.1V 이나 0.0.2V 에는 아직 0.0.3V 에는 사용하지 않는 메서드가 있다는 것이고 그 말은 버전을 하기 전에 가장 안정한 버전이라는 것입니다 만약 신규로 배포한 0.0.3V 가 오류가 발생하면 그 즉이 0.0.3V 도커 이미지 해제후 이전버전 0.0.2V 로 돌아가는것이 쉽다는 뜻입니다 이래서 태그를 사용합니다

## 오류 버전 종료 후 가장 최근 버전 up 
```
C:\Users\kimdongy1000>docker ps

CONTAINER ID   IMAGE                               COMMAND                CREATED         STATUS         PORTS                    NAMES
2173f4a235c3   proejct-spring-boot-docker:0.0.4V   "java -jar /app.jar"   6 seconds ago   Up 5 seconds   0.0.0.0:5000->8080/tcp   peaceful_visvesvaraya

C:\Users\kimdongy1000>docker stop peaceful_visvesvaraya

docker run -p 5000:8080 proejct-spring-boot-docker:0.0.3V

```

이렇게 하면 현재 진행중인 버전을 stop 하고 가장 최근의 tag 의 이름을 찾아서 run 을 하게 됩니다 이 이후에 도커 컴포즈가 있긴 하지만 그에 대해서는 











