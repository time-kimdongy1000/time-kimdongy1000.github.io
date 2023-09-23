---

title: Spring Secuirty 17 리소스 서버 구축 1
author: kimdongy1000
date: 2023-06-17 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

인가서버 Git 주소 : https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_Authentication_Server?ref_type=heads

리소스서버 Gti 주소 https://gitlab.com/kimdongy1000/spring_security_web/-/tree/ReSource_Server?ref_type=heads

우리는 지난시간에 인가서버를 구축했습니다 오늘은 리소스 서버를 구축하는것으로 통신을 해보도록 하겠습니다 

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
	<groupId>com.example</groupId>
	<artifactId>Spring-web-Resource-server</artifactId>
	<version>1.0.0</version>
	<name>Spring-web-Resource-server</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>11</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/com.google.code.gson/gson -->
		<dependency>
			<groupId>com.google.code.gson</groupId>
			<artifactId>gson</artifactId>
			<version>2.8.9</version>
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
인가서버는 간단하게 갈 예정입니다 시큐리티를 쓰지 않고 오롯이 web 과 gson 으로만 구성을 해서 인가서버와 통신을 해서 데이터를 주고 받을 예정입니다 


## JwtFilter
```

package com.example.demo.filter;

import com.example.demo.dto.JwtResponseDto;
import com.google.gson.Gson;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.util.StringUtils;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;


public class JwtFilter extends  OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        String jwtToken = request.getHeader("Authorization_token");

        if(!StringUtils.hasText(jwtToken)){

            RequestDispatcher dispatcher = request.getRequestDispatcher("/error/unAuthorization");
            dispatcher.forward(request, response);


            return;

        }else{

            String url = "http://localhost:8080/jwtParse";
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            Map<String , Object> map = new HashMap<>();
            map.put("jwtToken" , jwtToken);

            Gson gson = new Gson();

            String requestBody = gson.toJson(map);

            HttpEntity<?>  httpEntity = new HttpEntity<>(requestBody, headers);
            RestTemplate restTemplate = new RestTemplate();

            JwtResponseDto jwt_response = restTemplate.postForObject(url, httpEntity, JwtResponseDto.class);

            if(jwt_response.getStatusCode() == 200){

                filterChain.doFilter(request , response);
                return;

            }else{
                RequestDispatcher dispatcher = request.getRequestDispatcher("/error/unAuthorization");
                dispatcher.forward(request, response);
                return;
            }
        }
    }
}


```

JwtFilter 는 모든 요청이 들어오면 이 요청으로 들어와서 jwt 토큰 검증을 하게 됩니다 만약 토큰이 없으면 그냥 바로 탈락해서 에러 페이지로 인도합니다 

그리고 토큰이 있으면 `String url = "http://localhost:8080/jwtParse";` 주소와 통신을 시도해서 앞에 인가서버와 jwt 토큰과 토큰 결과를 받아서 

유효한 토큰이면 원래 인가서버 클라이언트 요청을 진행할것이고 아니면 마찬가지로 에러페이지 `RequestDispatcher dispatcher = request.getRequestDispatcher("/error/unAuthorization");` 인도하게 될것입니다 


## Filter 등록
```

package com.example.demo.config;

import com.example.demo.filter.JwtFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.filter.OncePerRequestFilter;

@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<OncePerRequestFilter> filterList(){
        FilterRegistrationBean<OncePerRequestFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new JwtFilter());
        registrationBean.addUrlPatterns("/*");
        return registrationBean;
    }
}


```

그리고 Filter 를 등록해서 웹 요청이 들어올떄 동작하게 됩니다 

## ErrorController

```

package com.example.demo.controller;

import com.google.gson.Gson;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("error")
public class ErrorController {

    @GetMapping("/unAuthorization")
    public ResponseEntity<?> unAuthorization(){

        Gson gson = new Gson();
        String result_error_message = gson.toJson("UnAuthorized");

        return new ResponseEntity<>(result_error_message , HttpStatus.UNAUTHORIZED);
    }
}


```

위에 Filter 에서 jwt 토큰이 없거나 , jwt 토큰이 만료된결과를 받으면 이쪽으로 돌아와서 인가서버 클라이언트에  `HttpStatus.UNAUTHORIZED` 에러를 던지게 됩니다 

## DemoController

```

package com.example.demo.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;


@Controller
@RequestMapping("demo")
public class DemoController {

    @GetMapping("")
    public ResponseEntity<?> demoHandler(){

        System.out.println("demoController");

        Map<String , Object> param = new HashMap<>();
        param.put("test" , "1");

        return new ResponseEntity<>(param , HttpStatus.OK);
    }
}


```

만약 유효한 토큰이면 원래 갈려고 했던 핸들러 demo 로 들어와서 원하는 리소스를 return 을 받게 됩니다 

## JwtResponseDto
```

package com.example.demo.dto;

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
인가서버와 마찬가지로 JwtResponseDto 를 사용해서 토큰의 상태를 확인할때 사용하는 Dto 입니다 


## CORS 등록 

```
package com.example.demo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("http://localhost:8080")
                .allowedMethods("GET" , "POST" , "PUT" , "PATCH" , "DELETE" , "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}


```

그리고 이 리소스 페이지는 좀 특이한 설정이 하나 필요합니다 이 addCorsMappings 입니다 사실 

인가서버 -> 리소스 서버로 요청을 넣을떄는 같은 host 이긴 하지만 포트가 서로 다르게 됩니다 이때 web 은 웹보안이라 해서 발생하는 문제인데 웹은 

기본적으로 동일 출처 정책 을 채택하게 됩니다 이는 동일 host , 동일 port 에서 오는 요청에 대해서만 자신의 데이터를 주게 되고 그렇지 않으면 기본적으로 

데이터를 가지고 오지 못하게 막는 웹정책입니다 그런 정책은 현재 요청이 오는 쪽에서 위와 같이 풀어주는 http://localhost:8080 설정하는 것으로 

풀게 되는것입니다 이런문제는 웹에서 -> 서버 호출할때 발생하는 것으로 서버에서 -> 서버로 요청하는것은 발생하지 않습니다 

자 그러면 우리는 인가서버 , 리소스 서버 모든것을 작성했습니다 이제 간단한 테스트를 통해서 확인을 해보겠습니다



![인가서버 , 리소스 서버 통신 성공](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/39f41306-2f9d-479d-9cd7-1a05ef62fd8a)

이렇게 해서 통신을 성공하게 됩니다 

그리고 우리는 jwt 토큰 설정을 1분으로 해놓았기 때문에 다시 그토큰으로 리소스를 호출하게 되면 

로그아웃 페이지로 나가게 됩니다 우리는 이렇게 인가서버와 리소스 서버 구축을 해보았고 이런 패턴은 다음시간부터 하게될 Oauth 와 매우 유사한 방식으로 만들어보았습니다

소스는 모두 업데이트 해놓을 예정입니다 


인가서버 Git 주소: https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_Authentication_Server?ref_type=heads

리소스서버 Gti 주소 https://gitlab.com/kimdongy1000/spring_security_web/-/tree/ReSource_Server?ref_type=heads





