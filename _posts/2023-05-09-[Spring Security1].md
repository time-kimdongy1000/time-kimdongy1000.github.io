---
title: Spring Secuirty Hello Spring Security
author: kimdongy1000
date: 2023-05-09 10:49
categories: [Back-end, Spring - Security ]
tags: [ Spring-Security ]
math: true
mermaid: true
---


# Spring Secuirty 란 
Spring Security is a powerful and highly customizable authentication and access-control framework. It is the de-facto standard for securing Spring-based applications.
스프링 시큐리티는 강력하고 높은 커스터마이징 가능한 인증 그리고 허가 컨트롤이 가능한 프레임워크 입니다. 스프링 기반의 어플리케이션들에 대한 보안의 기준이 됩니다.

Spring Security is a framework that focuses on providing both authentication and authorization to Java applications.
스프링 시큐리는 인증과 인가 둘다 집중한 java 어플리케이션 프레임워크입니다

이는 스프링 시큐리티 개요에 적혀 있는 소개글입니다 이 내용을 정리하면 시큐리티는 스프링 기반의 어플리케이션에서 강력한 인증 , 인가 와 관련한 프레임워크 제공과 사용자 입맛에 맞게 커스터마이징이 가능한 프레임워크라고 소개합니다 

# 환경설정 
우리는 앞으로 다음의 환경에서만 시큐리티를 다룰 예정입니다.

java version (opend-jdk) 11 
Spring boot 2.7.1
maven 

이런 형식으로 갈 예정이고 안에 maven 은 하단에 정리해서 쓰도록 하겠습니다 
```
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
</dependencies>
```

크게는 web , security 에 대해서만 의존성을 주입 후 여러가지 재미 있는것들을 해볼 예정입니다 


## 간단한 Controller 만들기 
```
@RestController
public class DemoController {
    
    @GetMapping("/demo")
    public String demo (){
        
        return "demo";
    }
}
```

우리는 이렇게 간단한 demo 핸들러를 만들고 실행을 해보자 

우리는 주소창에 http://localhost:8080/demo  입력하고 엔터를 치지만 아래 스크린샷처럼 demo 페이지가 나오는것이 아니라 지금과 같은 로그인 페이지가 나오게 됩니다 

![스크린샷 2023-08-06 105416](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/39942fb0-5695-4970-ab8e-50a4607773fb)

이런 설정은 시큐리티의 가장 기본적인 로그인 페이지입니다 그럼 로그인을 할려면 아이디 비밀번호가 있어야 하는데 아무런 설정없이 실행하는 경우에는 아이디 비밀번로를 
임의로 만들어서 발급을 해줍니다 


## admin 아이디 비밀번호

```
Using generated security password: aa5fb719-d2cf-46a9-9460-dee63292ca3d

This generated password is for development use only. Your security configuration must be updated before running your application in production.
```

이때 임시번호를 발급해주고 하단 문구에 이 비밀번호는 오로지 개발단계에서만 쓰라고 경고를 해줍니다 


![스크린샷 2023-08-06 105416](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/1b0257ff-98c3-4e30-80dd-9f8d42d7a082)

이렇게 우리가 만든 기본적인 페이지가 나오게됩니다 

아무런 설정이 없을때 시큐리티는 들어오는 모든 요청을 fitler 해서 권한이 없으면 로그인 페이지로 되돌리게 됩니다 이것이 가장 기본적인 스프링의 시큐리티의 인증 인가 어플리케이션입니다 우리는 앞으로 이런 시큐리티를 활용하면서 다양한 로그인 방법 및 시큐리티 프레임워크를 소개해 나가는 시간을 가질예정입니다