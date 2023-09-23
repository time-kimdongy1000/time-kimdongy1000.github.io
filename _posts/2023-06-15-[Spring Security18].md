---

title: Spring Secuirty 16 인가서버 구축
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
인가서버는 거의 그대로입니다 그리고 회원가입이나 , 로그인 로직같은 경우는 우리가 이제까지 해왔던 로직이기 떄문에 생략하겠습니다 GIT 소스를 참고해주세요 


## SecurityConfig 

```

package com.cybb.main.config;

import com.cybb.main.user.CustomAuthenticationProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private CustomAuthenticationProvider customAuthenticationProvider;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{


        httpSecurity.authorizeRequests().antMatchers("/"
                                                                ,"/signUp"
                                                                ,"/signUp/*"
                                                                ,"/bootStrap/css/**"
                                                                ,"/bootStrap/js/**"
                                                                ,"/resources/css/**"
                                                                ,"/resources/js/**"
                                                                ,"/jwtParse"


                ).permitAll()


                .anyRequest().authenticated();


        httpSecurity.formLogin()
                .loginPage("/login")
                .loginProcessingUrl("/login")
                .usernameParameter("userEmail")
                .passwordParameter("userPassword")
                .defaultSuccessUrl("/jwt")
                .permitAll();

        httpSecurity.logout()
                    .logoutUrl("/logout")
                    .logoutSuccessUrl("/login")
                    .invalidateHttpSession(true)
                    .clearAuthentication(true)
                    .deleteCookies("JSESSIONID");

        httpSecurity.csrf().disable();




        httpSecurity.authenticationProvider(customAuthenticationProvider);

        return httpSecurity.build();
    }
}


```

전체적인 SecurityConfig 는 올리겠습니다 여기서 크게 다른점은 `httpSecurity.csrf().disable();` 입니다 이는 서버간의 통신에서 서버끼리는 CSRF 토큰을 주고 받기가 어렵기 때문에 주로 서버끼리 통신에는 이 CRSF 토큰을 사용하지 않는 편입니다 

그리고 로그아웃을 설정했습니다 이 로그아웃을 할때 필요한것은 로그아웃 url , 로그아웃 성공시 오는 페이지 그리고 인증 및 인가 제거 쿠키 제거가 포함되어 있습니다 


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
            Long jwtTokenTime = Duration.ofMinutes(1).toMillis();

            String jwtToken = Jwts.builder()
                    .setSubject(userDetails.getUsername())
                    .setIssuedAt(now)
                    .setExpiration(new Date(now.getTime() + jwtTokenTime))
                    .signWith(SignatureAlgorithm.HS512, secretKey)
                    .compact();

            JwtDto jwtDto = new JwtDto();
            jwtDto.setJwtToken(jwtToken);

            return new ResponseEntity<>(jwtDto , HttpStatus.OK);

        }catch (Exception e) {

            throw new RuntimeException(e);
        }
    }


}


```

이 JwtController 은 로그인을 성공하게 되면 성공 페이지 및 Jwt 토큰을 발행하는 핸들러로 이루어졌습니다 토큰 만료시간은 1분으로 진행하겠습니다 


## JWTParseController
```

package com.cybb.main.controller;

import com.cybb.main.dto.JwtResponseDto;
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.SignatureException;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.Map;

@Controller
public class JWTParseController {

    @Value("${jwt.secret.key}")
    private String secretKey;

    @PostMapping("/jwtParse")
    public ResponseEntity<JwtResponseDto> jwtParse(@RequestBody Map<String , Object> param)
    {

        JwtResponseDto jwtResponseDto = new JwtResponseDto();
        try{

            String jwtToken = (String)param.get("jwtToken");



            if(!StringUtils.hasText(jwtToken)){

                jwtResponseDto.setJwtFlag(false);
                jwtResponseDto.setMessage("JWT 토큰이 없습니다");
                jwtResponseDto.setStatusCode(401);

                return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);
            }

            Claims claims = Jwts.parserBuilder()
                    .setSigningKey(secretKey)
                    .build()
                    .parseClaimsJws(jwtToken)
                    .getBody();

            jwtResponseDto.setJwtFlag(true);
            jwtResponseDto.setMessage("인증성공");
            jwtResponseDto.setStatusCode(200);

            return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);

        }catch(ExpiredJwtException e1){

            jwtResponseDto.setJwtFlag(false);
            jwtResponseDto.setMessage("JWT 인증시간 만료");
            jwtResponseDto.setStatusCode(401);

            return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);

        }catch (SignatureException e2){

            jwtResponseDto.setJwtFlag(false);
            jwtResponseDto.setMessage("개인키 오류");
            jwtResponseDto.setStatusCode(401);

            return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);

        }catch (UnsupportedJwtException e3){

            jwtResponseDto.setJwtFlag(false);
            jwtResponseDto.setMessage("개인키 오류");
            jwtResponseDto.setStatusCode(401);

            return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);

        }catch (MalformedJwtException e3){

            jwtResponseDto.setJwtFlag(false);
            jwtResponseDto.setMessage("JWT 형식 오류");
            jwtResponseDto.setStatusCode(401);

            return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);

        }catch (Exception e){

            jwtResponseDto.setJwtFlag(false);
            jwtResponseDto.setMessage(e.getMessage());
            jwtResponseDto.setStatusCode(401);

            return new ResponseEntity<>(jwtResponseDto , HttpStatus.OK);

        }

    }
}

```
JWTParseController 은 전체접근이 가능한 페이지로 리소스 서버가 JWT 유효성 검증을 위해서 통신하기 위한 핸들러이빈다 리소스 서버 작성할때 이곳과 통신하는 소스를 작성할 예정입니다 

```

package com.cybb.main.dto;

public class JwtResponseDto {

    private boolean jwtFlag;

    private String message;

    private int statusCode;

    public int getStatusCode() {
        return statusCode;
    }

    public void setStatusCode(int statusCode) {
        this.statusCode = statusCode;
    }

    public boolean isJwtFlag() {
        return jwtFlag;
    }

    public void setJwtFlag(boolean jwtFlag) {
        this.jwtFlag = jwtFlag;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}


```

JwtResponseDto 는 리소스 서버와 인가서버 동시에 쓰는 Jwt 와 관련한 정보입니다 여기에는 jwt 상태와 , 에러메세지 등을 담을 예정입니다 


## jwt.html 
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
<button id = "btn_getResource" type="button" class="btn btn-primary">리소스 가져오기</button>

<form action="/logout" method="post">

    <!--<input  th:name="${_csrf.parameterName}" th:value="${_csrf.token}">-->
    <button id = "btn_logout" class="btn btn-primary" type="submit" >로그아웃</button>
</form>


<input  id = "jwt_input">






<script src="/bootStrap/js/bootstrap.bundle.min.js"></script>
<script src="/resources/js/jwt.js"></script>
</body>
</html>


```

로그인 성공하면 제일 먼저 들어오는 페이지입니다 이 페이지에서 jwt 를 발급받고 , 리소스 서버의 데이터를 주고 받는 페이지입니다


## jwt.js
```

const jwt_button = document.querySelector("#btn_generate_jwt");
const csrf_input = document.querySelector('input[name="_csrf"]');
const jwt_input = document.querySelector("#jwt_input");
const btn_getResource = document.querySelector("#btn_getResource");
const btn_logout = document.querySelector("#btn_logout")


jwt_button.addEventListener('click' , (e) => {

    alert('jwt 발급 버튼 클릭')

    const domain = "http://localhost:8080";
    const api_url = "/jwt/generate";
    const method = "post";
    //const csrf_token = csrf_input.value;

    try{

        fetch(domain + api_url , {
            method : method ,
            headers : {
                  "Content-Type": "application/json" ,
                   /*"X-CSRF-Token" : csrf_token*/
           },


        }).then( (res) => res.json())
          .then( (data) => {
            jwt_input.value = data.jwtToken;
          })

    }catch(error){
        console.log(error)
    }

})

btn_getResource.addEventListener('click' , (e) => {

    const domain = "http://localhost:8090";
    const api_url = "/demo";
    const method = "GET";

    try{

        fetch(domain + api_url , {
            method : method ,
            headers : {
                "Authorization_token" : jwt_input.value
           },


        }).then( (res) => {

            if(res.status == 401){
               btn_logout.click();
            }else{
                 return res.json();
            }
        }).then( (data) => {
            console.log(data);
        })

    }catch(error){
        console.log(error)
    }

})

```

이 jwt 는 jwt.html 연결된 페이지로 인가서버에서 받은 jwt 토큰과 리소스 서버 연결을 위한 페이지입니다 리소스 가져오기 버튼을 누르면 리소스 서버와 통신해서 
정보를 가져오거나 , 아니면 jwt 토큰이 없거나 , 미인증 토큰이면 로그아웃이 되게끔 설계 했습니다 
