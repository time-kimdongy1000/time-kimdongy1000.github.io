---

title: Spring Secuirty 16 인가서버와 , 리소스 서버 구축하기 1
author: kimdongy1000
date: 2023-06-15 10:00
categories: [Spring, Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

지난시간까지 JWT 하다가 갑자기 왜 인가서버와 , 리소스 서버 구축하기로 변경이 되었냐면 이 주제를 할려고 앞에서 JWT 발급과 검증과정에 대해서 공부를 했다 
이번 파트는 또 개인적으로 만들고 싶은것을 한번 만들어볼 예정이다 이때는 확실히 자신의 책임에 맞는 서버를 구축할것이다 

## 인가서버 
앞으로 인가서버는 프로그램에 로그인을 담당할 것이며 로그인에 성공하면 JWT 를 발급해주고 리소스 서버가 던지는 JWT 에 대해서 검증과 유효성을 판단하는 서버를 만들것이다 

## 리소스 서버 
앞으로 리소스 서버는 로그인 화면은 없습니다 클라이언트는 JWT 토큰을 통해서 자신의 권한을 전달하면 그에 맞는 자원을 조회 할 수 있는 서버를 만들것입니다 


## 내가 생각하는 프로세스 Flow

그럼 구체적으로 프로세서 flow 를 살펴보면 

1) 클라이언트는 리소스 서버에 자원을 요청합니다 

2) 리소스 서버는 클라이언트 서버에서 던진 access_token (jwt) 유무를 확인합니다 

3) jwt 가 없으면 인가서버로 redirect 를 해서 인가서버에 로그인을 하라고 지시합니다 

4) 클라이언트가 redirect 된 인가서버에서 로그인을 완료하면 인가서버는 해당 유저에 맞는 jwt 를 발급합니다 

5) 클라이언트는 다시 이 jwt 를 가지고 리소스 서버에 자원을 요청합니다 

6) 리소스 서버는 jwt 토큰 vaild 를 인가서버로 던져서 발급 및 유효여부를 판단합니다 

7) jwt 유효 여부를 확인한 후 리소스 서버는 적절한 행동을 취합니다 (유효 요청한 자원 발급 , 무효 다시 인가서버의 로그인 화면 요청)


이렇게 진행이 될꺼 같습니다 

그럼 총 2개의 서버가 만들어질것입니다 먼저 인가서버를 구축하고 리소스서버를 구축하겠습니다 


## 인가서버 maven 

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
	<artifactId>SpringBoot_web_Security</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>SpringBoot_web_Security</name>
	<description>Project_Amadeus</description>
	<properties>
		<java.version>11</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-api -->
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-api</artifactId>
			<version>0.11.5</version>
		</dependency>

		<!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-impl -->
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-impl</artifactId>
			<version>0.11.5</version>
			<scope>runtime</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-jackson -->
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-jackson</artifactId>
			<version>0.11.5</version>
			<scope>runtime</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-jpa -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>


		<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>

		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
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

인가서버는 사용자 정보를 저장하는 jpa와 jwt 를 발급하는 시스템을 만들고 jwt 를 발급 해서 클라이언트에 심는 로직을 만들겠습니다 
화면이나 자바 스크립트는 지난시간에 만들었던 커스텀 회원가입을 최대한 따라갈 예정입니다 



