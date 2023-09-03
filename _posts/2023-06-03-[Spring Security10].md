---

title: Spring Secuirty 10 나만의 로그인 및 회원가입 만들기 
author: kimdongy1000
date: 2023-06-04 14:00
categories: [Spring, Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---


브랜치 주소 https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_0903

소스 공유 https://gitlab.com/kimdongy1000/spring_security_web/-/commit/bbf6e35fbbb430194e04dda88c241b5be907fd50



이제까지 우리는 시큐리티가 기본적으로 제공하는 로그인 페이지 로그인 화면 이런것들을 사용했다 이제는 직접 한번 간단하게 만들어보는 시간을 가질것이다 
직접 회원가입부터 로그인했을때 db 를 불러와서 인증을 시키는거 까지 해볼예정이다 

랜더링 화면은 타임리프로 구성할 것이고 db 는 jpa 로 구성을 할것이다 jpa 는 나도 잘 안해보았지만 간단한 db를 구축할때는 자주자주 사용하는 편이다 
그럼 maven 을 구성을 해보자 

Branch 명 main_0903

브랜치 주소 https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_0903

소스 공유 https://gitlab.com/kimdongy1000/spring_security_web/-/commit/2af7030b2285bd703a5f3ba2762ea9feac67940a


## maven 구성 
```

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

```



## css 플랫폼은 bootstrap 5 사용
그리고 우리는 bootstrap 를 사용할것이다 이때 필요한 파일은 
https://github.com/twbs/bootstrap/releases/download/v5.3.1/bootstrap-5.3.1-dist.zip 다운로드 받으면 알집안에 css 와 , js 파일이 있는데 그중에서

css 는 bootstrap.min.css 를 사용할것이고 js는 bootstrap.bundle.min.js 를 사용할것입니다 그럼 파일 위치는 아래 사진처럼 두시면됩니다 



## 디렉토리 구성 
![파일 위치](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ecab4d7f-80e0-425e-8758-d0f5dd7f3c8c)





그럼 회원가입 핸들러 및 회원가입 페이지를 간단하게 만들어보겠습니다 


## 회원가입 핸들러 SignUpController
```

@Controller
@RequestMapping("signUp")
public class SignUpController {

    @GetMapping("/")
    public String signUpPage(){

        return "signUp";
    }
}


```

그럼 당연히 우리는 앞에서 배운 SecurityFilterChain 에서 이 회원가입 페이지는 인증받지 않는 유저도 접속을 허용을 해야 합니다 그러면 

## 미인증 유저도 들어올 수 있는 filter 작성

```

@Bean
public SecurityFilterChain  securityFilterChain(HttpSecurity httpSecurity) throws Exception{

    httpSecurity.authorizeRequests().antMatchers("/" , "/signUp/*").permitAll()
                    .anyRequest().authenticated();

    httpSecurity.formLogin();

    return httpSecurity.build();
}


```
`httpSecurity.authorizeRequests().antMatchers("/" , "/signUp/*").permitAll().anyRequest().authenticated();` 

이 전체가 한문장입니다 이때 /  , /signUp/* 이런 패턴으로 들어오는 요청에 대해서는 미인증 유저가 들어올 수 있는 permitAll 을 사용한것입니다 그리고 그 외 요청은 일단은 인증유저만 들어 올 수 있게 해놓겠습니다 

## signUp.html 작성 

```

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <link href="/bootStrap/css/bootstrap.min.css" rel="stylesheet">

    <link href="/resources/css/register.css" rel="stylesheet">

    <title>회원가입 페이지</title>
</head>
<body>


<main class="form-signin">
    <form>
        <h1 class="h3 mb-3 fw-normal"> 회원가입 </h1>

        <div class="form-floating">
            <input type="email" class="form-control" id="floatingInput" placeholder="name@example.com">
            <label for="floatingInput">Email address</label>
        </div>
        <div class="form-floating">
            <input type="password" class="form-control" id="floatingPassword" placeholder="Password">
            <label for="floatingPassword">Password</label>
        </div>
        <div class="form-floating">
            <input type="password" class="form-control" id="floatingPassword_confirm" placeholder="Password_confirm">
            <label for="floatingPassword_confirm">Password_confirm</label>
        </div>


        <button id = "btn_register" class="w-100 btn btn-lg btn-primary" type="button"> 회원가입 신청 </button>
    </form>
</main>






<script src="/bootStrap/js/bootstrap.bundle.min.js"></script>
</body>
</html>


```

이렇게 작성을 하고 저장을 하고 이 페이지로 렌더링을 시도하겠습니다 하지만 화면이 전혀 렌더링 되지 않습니다 태그만 잔뜩 붙어 있는 이상한 화면이 생기게 되는데 
![자원들도 302 걸립니다](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f37551a4-c029-4bc3-a03e-48e2e3639f09)

지금보면 우리가 사용할려고 하는 스크립트 , css 도 전부 filter 에 걸리게 됩니다 이런 filter 들도 전부 접근 가능하게 해주어야 합니다 

## WebConfig 

```
package com.cybb.main.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/bootStrap/css/**").addResourceLocations("classpath:/static/bootstrap/css/");
        registry.addResourceHandler("/bootStrap/js/**").addResourceLocations("classpath:/static/bootstrap/js/");

        registry.addResourceHandler("/resources/css/**").addResourceLocations("classpath:/static/resources/css/");
        registry.addResourceHandler("/resources/js/**").addResourceLocations("classpath:/static/resources/js/");
    }
}


```

이 파일은 mvc 에서 자원을 패턴을 관리하는 소스인데 저는 자원관리를 효율적으로 하기 위해서 기존 정의된것 말고 새롭게 정의해서 사용할 예정입니다 

그리고 FilterChain 으로 돌아와서 
`httpSecurity.authorizeRequests().antMatchers("/" , "/signUp/*" , "/bootStrap/css/**" , "/bootStrap/js/**" , "/resources/css/**","/resources/js/**").permitAl().anyRequest().authenticated();`

이렇게 추가된 부분을 재작성 해주면 이제 자원관련된 부분도 허용이 되었습니다 

## register.css 

```

html,
body {
  height: 100%;
}

body {
  display: flex;
  align-items: center;
  padding-top: 40px;
  padding-bottom: 40px;
  background-color: #f5f5f5;
}

.form-signin {
  width: 100%;
  max-width: 330px;
  padding: 15px;
  margin: auto;
}

.form-signin .checkbox {
  font-weight: 400;
}

.form-signin .form-floating:focus-within {
  z-index: 2;
}

.form-signin input[type="email"] {
  margin-bottom: -1px;
  border-bottom-right-radius: 0;
  border-bottom-left-radius: 0;
}

.form-signin input[type="password"] {
  border-top-left-radius: 0;
  border-top-right-radius: 0;
}

.form-signin input[type="text"] {
  margin-top: 20px;
  border-top-left-radius: 0;
  border-top-right-radius: 0;
}


```

로그인 페이지를 좀더 이쁘게 만들기 위해서 다음과 같은 css 를 추가하겠습니다 그러면 이와 같은 화면이 나오게 되는데 

![파일 위치](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/70597fc7-b949-441c-8922-8447cf54ca0c)

이렇게 나오게 됩니다 이번 포스터는 기초작업 + 설정 및 회원가입 form 만드는거 까지만 진행을 하고 다음시간에 회원가입 핸들러 및 db 저장에 대해서 해보겠습니다 
모든 소스는 공유할 예정입니다 

https://gitlab.com/kimdongy1000/spring_security_web/-/commit/2af7030b2285bd703a5f3ba2762ea9feac67940a





