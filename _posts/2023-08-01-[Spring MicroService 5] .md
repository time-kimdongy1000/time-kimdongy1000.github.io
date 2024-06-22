---
title: Spring MicroService 5 Spring MicroService 와 Docker - Compose
author: kimdongy1000
date: 2023-08-01 11:30
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService , Docker ]
math: true
mermaid: true
---

지난시간에는 docker 와 docker-compose 를 왜 사용해야 하는지에 대해서 알아보았습니다 이번시간에는 지난시간 docker-compose 의 명령어를 해석하면서 진행을 해보도록 하겠습니다 

## 도커 컴포즈 스크립트 

```
version: '3.8'

services:
  db:
    image: mysql:5.7
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  app:
    build: .
    container_name: app
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/mydb
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root_password

volumes:
  db_data:

```

```
version : 
도커 컴포즈 도구의 버전을 지정합니다 버전마다 지원하는 스크립트가 다르거나 추가된 내용이 있습니다 

services : 
  배포할 서비스를 지정합니다 즉 이곳에서는 여러 컨테이너를 정의하는 섹션입니다 이곳에서는 db (mysql) , app(webApplication) 을 정의하고 있습니다 

image: 
  특정 이미지를 사용하여 컨테이너를 실행하도록 지정합니다 이때 mysql:5.7 이 로컬 컨테이너에 존재하지 않으면 도커 허브에서 해당 버전을 pull 해서 로컬 컨테이너에 설치를 하게 됩니다 

container_name : 
  해당 컨테이너 이름을 지정할 수 있습니다 

environment : 
  해당 컨테이너 환경을 지정할 수 있습니다 이때 mysql 의 환경으로는 지정한 옵션들은 root 패스워드 , 생성할 dataBase , 생성할 user , 생성된 유저의 비밀번호 

port: 
  해당 컨테이너가 외부에 노출할 포트와 내부에 매핑될 포트를 적어둡니다 [외부노출 포트] : [내장 매핑 포트] 포트포워딩 작업이라고 생각하면 됩니다 

volumes : 
  호스트의 db_data 볼륨을 컨테이너의 /var/lib/mysql에 매핑합니다. 이 말이 무엇이냐면 도커 컨테이너에서 volumes 설정을 사용하면 호스트 시스템과 컨테이너 간에 데이터 공유할 수 있는 옵션입니다 데이터베이스의 데이터 디렉토리를 볼륨으로 지정하여 데이터베이스 컨테이너가 재시작되거나 업데이트 되더라도 보존처리 할 수 있습니다 

app : 
  도커 컴포즈의 service 중 db 말고 app 또한 있다고 명시를 합니다 

build : 
  디렉토리에 존재한 Dockerfile 을 사용하여 이미지 빌드합니다 현재 . 이라고 적혀 있기 때문에 프로젝트 root Dockerfile 을 스크립트로 사용하여 빌드 하는 것입니다 

depends_on : 
  이는 의존을 나타냄으로 db 컨테이너가 실행이 되고 난 뒤에 app 서비스가 시작되게 해두었습니다 

environment : 
    해당 컨테이너 환경을 지정할 수 있습니다 이때 웹 어플리케이션의 옵션으로 SPRING_DATASOURCE_URL , SPRING_DATASOURCE_USERNAME , SPRING_DATASOURCE_PASSWORD 를 사용합니다 이때 application.yml , docker-compose 에 각각 DB 에 관련한 정보를 적어 두었는데 이때 우선순위는 docker-compose 가 우선됩니다 
```    

## application.yml VS docker-compose
위에서 application.yml  을 정의해 두어도 실제로는 docker-compose 가 해당 내용의 값을 대체하게 됩니다 우리는 앞의 정보에서 application.yml 에 DB 정보를 도커 컨테이저 정보로
매핑을 해두었습니다 그렇게 할지라도 실제 도커 컴포즈는 docker-compose.yml 파일을 우선하여 application.yml 파일을 대체해서 정보를 주입하게 됩니다 그 정보는 

```
SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/mydb
SPRING_DATASOURCE_USERNAME: root
SPRING_DATASOURCE_PASSWORD: root_password

```

이렇게 정의한 root 의 정보가 주입되게 됩니다 









  

