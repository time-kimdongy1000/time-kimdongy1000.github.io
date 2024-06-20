---
title: Spring MicroService 4 Spring MicroService 와 Docker - Compose
author: kimdongy1000
date: 2023-08-01 11:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService , Docker ]
math: true
mermaid: true
---

지난시간에는 Docker 와 Dockerfile 그리고 안에 있는 명령어를 정리했습니다 이번엔 도커 컴포즈에 대해서 알아보도록 하겠습니다 
그 전에 Docker - Compose 를 하기 전에 Dokcer 로 mysql 이미지를 만들어서 한번 띄어보도록 하겠습니다 이것을 왜 하는지에 대해서는 중간에 설명을 하도록 하겠습니다 

## mysql 이미지 받기 
mysql 은 도커 허브에서 올려준 파일로 바로 받을 수 있습니다 

```
docker pull mysql :latest

```

이렇게 하면 최신버전의 mysql 을 받을 수 있습니다 

![docker_compose_1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/5fb24804-da65-48a0-9028-df5fee3d1265) 

그러면 이렇게 mysql 이미지가 생겼습니다 그럼 이제 이 도커 이미지를 컨테이너로 실행을 하겠습니다 

## mysql 도커 컨테이너 실행 

```

docker run --name mysql -e MYSQL_ROOT_PASSWORD=1111111111 -d -p 4000:3306 mysql:latest

```

도커 mysql 도커 컨테이너를 실행하고 이때 root 계정으로 비밀번호는 1111111111 설정할 수 있고 4000번으로 들어오는 포트를 3306 으로 포트포워딩 해주고 있습니다 

그리고 그 내용을 DB 툴로 연결 할 수 있습니다 


![docker_compose_2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/b562b668-201c-4b82-8f00-9f32ba7a17bb)

이렇게 하면 mysql 도커 컨테이너로 이용해서 접속을 할 수 있습니다 


## 그럼 위의 예제를 보여주는 이유
우리 EE 급 어플리케이션을 개발할때 DB 접속은 필수입니다 그래서 Docker 가 나오기 전에는 일일이 개인 PC 에 설치를 하거나 또는 DB 서버 컴퓨터에 로 접속을 하게 됩니다 
하지만 이 Docker 이미지를 활용해서 DB 컨테이너를 만들어서 굳이 설치하거나 그럴 필요가 없어진것입니다 그럼 이 DB 컨테이너를 활용해서 우리 웹어플리케이션에 
DB를 연동해서 컨테이너로 개발을 해보도록 하겠습니다 

## 도커 컴포즈 
도커 컴포즈(Docker Compose)는 여러 개의 도커 컨테이너를 정의하고 실행하기 위한 도구입니다. 주로 복잡한 멀티 컨테이너 애플리케이션을 쉽게 관리하고 실행할 수 있도록 도와줍니다. 
즉 우리는 이를 이용해서 웹어플리케이션하고 DB 를 같이 기동하기 위해 도커 컴포즈를 이용할것입니다 


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
	<groupId>com.demo</groupId>
	<artifactId>Project-Spring-Boot-Docker-Compose</artifactId>
	<version>1.0.0</version>
	<name>Project-Spring-Boot-Docker-Compose</name>
	<description>Project_Amadeus</description>
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
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>2.7.1</version>
		</dependency>

		<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
		<dependency>
		    <groupId>mysql</groupId>
		    <artifactId>mysql-connector-java</artifactId>
		    <version>8.0.29</version>
		</dependency>
				
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
			<!-- https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter -->
	<dependency>
	    <groupId>org.mybatis.spring.boot</groupId>
	    <artifactId>mybatis-spring-boot-starter</artifactId>
	    <version>2.2.2</version>
	</dependency>

		<!-- https://mvnrepository.com/artifact/org.apache.maven.plugins/maven-resources-plugin -->
		<dependency>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-resources-plugin</artifactId>
			<version>3.3.1</version>
		</dependency>

	</dependencies>

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

이번 maven 은 통째로 넣어보겠습니다 크게 web 하고 mysql 하고 mybatis - starter 도 넣었습니다 


## application.properties 

```

spring.datasource.url=jdbc:mysql://localhost:3306/mydb?characterEncoding=UTF-8&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=root_password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver



```

## MySqlConfig
```
@Configuration
public class MySqlConfig {

    @Value("${spring.datasource.url}")
    private String url;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    @Bean("mysqlHikariConfig")
    public HikariConfig mysqlHikariConfig(){

        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(url);
        hikariConfig.setUsername(username);
        hikariConfig.setPassword(password);

        return hikariConfig;

    }

    @Bean("mysqlDataSource")
    public DataSource mysqlDataSource(HikariConfig mysqlHikariConfig){
        DataSource mysqlDataSource = new HikariDataSource(mysqlHikariConfig);
        return mysqlDataSource;
    }

    @Bean("mysqlSessionFactory")
    public SqlSessionFactory mysqlSessionFactory(DataSource mysqlDataSource) throws Exception{

        SqlSessionFactoryBean mysqlSessionFactoryBean = new SqlSessionFactoryBean();
        mysqlSessionFactoryBean.setDataSource(mysqlDataSource);
        mysqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/static/mapper/mysql/*.xml"));

        return mysqlSessionFactoryBean.getObject();

    }

    @Bean("mysqlSessionTemplate")
    public SqlSessionTemplate mySqlsessionTemplate(SqlSessionFactory mySqlSessionFactory) throws  Exception{
        SqlSessionTemplate mysqlSessionTemplate = new SqlSessionTemplate(mySqlSessionFactory);
        return mysqlSessionTemplate;
    }

}

```

HikariCP 가 자동으로 바인딩 하지만 저는 그래도 늘 이렇게 bean 으로 바인딩을 해줍니다 

## MySqlController
```
@RestController
public class MySqlController {

    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;

    @GetMapping("/getMysql")
    public ResponseEntity<Integer> getMysql()
    {


        return new ResponseEntity<>(sqlSessionTemplate.selectOne(this.getClass().getName() + ".getMysql") , null , HttpStatus.OK);
    }

    @GetMapping("/getMysql2")
    public ResponseEntity<Integer> getMysql2()
    {


        return new ResponseEntity<>(sqlSessionTemplate.selectOne(this.getClass().getName() + ".getMysql2") , null , HttpStatus.OK);
    }
}


```

이렇게 핸들러를 작성합니다 2개의 핸들러를 작성하고 이 2개의 핸들러를 mybatis 로 불러올 것입니다 

## xml 작성 

```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.cybb.main.controller.MySqlController">

    <select id="getMysql" resultType="Integer">

        SELECT 1 + 2 FROM DUAL
    </select>

    <select id="getMysql2" resultType="Integer">

        SELECT 3 + 4 FROM DUAL
    </select>

</mapper>

```

## Dockerfile

```

FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package -DskipTests

FROM openjdk:11-jdk-slim

COPY --from=build /app/target/*.jar /mysql_docker.jar

ENTRYPOINT ["java","-jar","/mysql_docker.jar"]


```

도커파일을 만들어줍니다 

## Docker-compose.yml 

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

설명은 다음 장에서 하겠습니다 오늘은 이렇게 하고 바로 실행을 해보겠습니다 

## 도커 컴포즈 실행 

```

docker-compose up --build

```

그러면 이제 도커 엔진은 다음과 같이 보일것입니다 

![docker_compose_3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0313900c-727b-4a4e-a042-c404e14fe763)

이렇게 도커 컴포즈 안에 app 와 db 가 같이 기동되는 모습을 볼 수 있을 것입니다 우리는 이렇게 해서 DB 를 로컬이나 다른 서버에 연동을 하지 않고도 웹 어플리케이션을 올려두었습니다 

## POSTMAN

![docker_compse_4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/84fd5f64-2f7b-4ad5-8c09-043b4b567a0d)

이렇게 웹어플리케이션에 요청을 하면 요청 정보가 DB 를 타고 들어오게 됩니다 쿼리는 간단하게 작성을 했지만 Bean 을 작성하고 컨테이너 DB 를 연결해서 
진행했습니다 

## 도커 컴포즈 중지 

```

docker-compose down

```

실행된 도커 컴포즈를 중단하기 위해선 이와 같은 명령어로 중단합니다 




