---

title: Spring Secuirty 16 인가서버와 , 리소스 서버 구축하기 1
author: kimdongy1000
date: 2023-06-15 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

지난시간까지 JWT 하다가 갑자기 왜 인가서버와 , 리소스 서버 구축하기로 변경이 되었냐면 이 주제를 할려고 앞에서 JWT 발급과 검증과정에 대해서 공부를 했다 
이번 파트는 또 개인적으로 만들고 싶은것을 한번 만들어볼 예정이다 이때는 확실히 자신의 책임에 맞는 서버를 구축할것이다 

## Git 주소 
https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_Authentication_Server?ref_type=heads


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
화면이나 자바 스크립트는 지난시간에 만들었던 커스텀 회원가입을 최대한 따라갈 예정입니다 그래서 전체소스는 gitlab 주소를 명시해서 보여줄 예정이지만 
이전에 했던 방식이니 html 생성이나 이런것들은 pass 하도록 하겠습니다 

## 로그인 성공시 바로 JWT 발급페이지로 이동 

```

httpSecurity.formLogin()
		.loginPage("/login")
		.loginProcessingUrl("/login")
		.usernameParameter("userEmail")
		.passwordParameter("userPassword")
		.defaultSuccessUrl("/jwt/getJwt")
		.permitAll();

```

나는 앞에서 defaultSuccessUrl 이 메서드를 사용햇는데 이 메서드는 로그인이 성공하면 기본적으로 옮겨지는 핸들러를 호출하게 됩니다 이때 핸들러를 정의하게 되면 
로그인 성공시 바로 핸들러를 호출하게 됩니다 이떄는 success 핸들러 뿐만 아니라 실패 핸들러 두개가 존재합니다
`defaultSuccessUrl , .defaultSuccessUrl` 이렇게 호출할 수 있으며 우리는 이 defaultSuccessUrl 를 이용해서 로그인 성공하면 바로 
jwt 발급해서 나의 헤더에 넣는 작업을 진행하도록 하겠습니다 

## JwtController

```

package com.cybb.main.controller;

import com.cybb.main.dto.JwtDto;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import java.time.Duration;
import java.util.Date;

@Controller
@RequestMapping("jwt")
public class JwtController {

    @Value("${jwt.secret.key}")
    private String secretKey;

    @GetMapping("")
    public String jwtPage(){

        return "jwt";
    }

    @PostMapping("/generate")
    public ResponseEntity<?> generateJwt (@AuthenticationPrincipal UserDetails userDetails){

        try {

            Date now = new Date();
            Long jwtTokenTime = Duration.ofMinutes(30).toMillis();

            String jwtToken = Jwts.builder()
                    .setSubject(userDetails.getUsername())
                    .setIssuedAt(now)
                    .setExpiration(new Date(now.getTime() + jwtTokenTime))
                    .signWith(SignatureAlgorithm.HS512, secretKey)
                    .compact();

            JwtDto jwtDto = new JwtDto();
            jwtDto.setJwtToken(jwtToken);

            return new ResponseEntity<>(jwtDto , HttpStatus.OK);

        }catch (Exception e){

            throw new RuntimeException(e);
        }
    }
}


```

우리는 여기서 jwt 발급 화면을 만드는 것이다 이전하고 완전 똑같긴 하지만 그래도 실제로 시크릿 key 같은 경우는 소스에 직접적인 노출을 막기 위해서 다양한 방법을 사용한다 
일단 일차적으로는 바로 보이는 곳에는 두지 않는다는 점이다 그렇다고 이 방법도 안전한것은 아니다 일단 우리는 applition.properties 에 이 key 를 넣고 불러와서 진행을 하겠습니다 

## applicaiton.properties

```

jwt.secret.key=cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3p


```

이렇게 세팅을 해주게 됩니다 

## jwt.html  , jwt.js

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <link href="/bootStrap/css/bootstrap.min.css" rel="stylesheet">



    <title>jwt 발급페이지 </title>
</head>
<body>

<button id = "btn_generate_jwt" type="button" class="btn btn-primary">jwt 발급받기</button>

<input  th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
<input  id = "jwt_input">






<script src="/bootStrap/js/bootstrap.bundle.min.js"></script>
<script src="/resources/js/jwt.js"></script>
</body>
</html>



```


```

const jwt_button = document.querySelector("#btn_generate_jwt");
const csrf_input = document.querySelector('input[name="_csrf"]');
const jwt_input = document.querySelector("#jwt_input");


jwt_button.addEventListener('click' , (e) => {

    alert('jwt 발급 버튼 클릭')

    const domain = "http://localhost:8080";
    const api_url = "/jwt/generate";
    const method = "post";
    const csrf_token = csrf_input.value;

    try{

        fetch(domain + api_url , {
            method : method ,
            headers : {
                  "Content-Type": "application/json" ,
                   "X-CSRF-Token" : csrf_token
           },


        }).then( (res) => res.json())
          .then( (data) => {
            jwt_input.value = data.jwtToken;
          })

    }catch(error){
        console.log(error)
    }




})


```

화면은 간단하게 post 을 통해서 jwt 를 발급받으면 이제 이 jwt 를 헤더에 심어서 다음에 작성할 리소스 서버에 전달할 예정입니다 물론 지금은 진짜 jwt 발급되어서 

보내지는것을 input 에 담아놓고 다음 요청때 이 input 값을 헤더에 심어서 보낼거지만 실제로는 이 jwt 또한 노출되어도 크게상관은 없지만 어찌되었든 

해당 사용자를 식별하는데 사용되는 토큰임으로 이렇게 input 에 대놓고 쓰지는 않습니다 이렇게 까지 하면 우리는 회원가입부터 JWT 발급까지 진행을 했고 

다음 포스터는 이제 리소스 서버를 작성을 해서 통신을 하는 과정을 그려보겠습니다 







