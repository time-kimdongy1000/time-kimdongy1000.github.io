---
title: Spring MVC @ModelAttribute
author: kimdongy1000
date: 2023-03-22 14:37
categories: [Back-end, Spring - MVC]
tags: [ MVC , ModelAttribute]
math: true
mermaid: true
---

## Spring MVC

Spring Web MVC 는 Servlet Api 를 기반으로 구축된 최초의 웹 프레임워크이며 처음부터 Spring 프레임워크에 포함되어 었습니다 
Spring 5.0 은 Spring WebFlux 는 반응형 스택 웹 프레임워크 입니다 

## maven

```

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

```

이번장에서 부터는 Web 에 대해서 공부를 해볼것입니다 Web 을 하기 전에 DispatchServlet 를 공부해야 하지만 내용도 많고 양도 방대하기 때문에 건너띄고 실무에서 
코딩하는 방법으로 직접 페이지를 하나하나 만들어가면서 진행을 하도록 하겠습니다 

![mvcProject](https://user-images.githubusercontent.com/58513678/227852147-9c979967-2dea-43b8-bfb4-0ba58cb1a72d.jpg)

프로젝트 구조는 이렇게 될것이고 간단한 html 을 작성하겠습니다 

## hello.html 

```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project

</body>
</html>

```


## HelloController.java 


```
package com.cybb.main.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;


@Controller
public class HelloController {

    @GetMapping("/hello")
    public String helloWorld(){

        return "hello";
    }
}

```
기본적으로 spring-boot-starter-thymeleaf 의존성을 걸게 되면 템플릿엔진이 알아서 prefix 와 surfix 를 알아서 설정한다 기본설정을 먼저 보고 진행을 하자
prefix=/templates/
surfix=.html 

이렇게 가게 된다 그래서 기본적인 controller 을 설정하면 웹 화면이 렌더링 되어서 나오게 된다 그럼 다른 방법으로 이번엔 렌더링 되는 위치를 바꾸어보자 

## SpringResourceTemplateResolver.java

```

package com.cybb.main.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.thymeleaf.spring5.templateresolver.SpringResourceTemplateResolver;

@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public SpringResourceTemplateResolver viewResolver(){

        SpringResourceTemplateResolver templateResolver = new SpringResourceTemplateResolver();

        templateResolver.setPrefix("classpath:/templates/view/");
        templateResolver.setSuffix(".html");

        return templateResolver;
    }
}

```

위와 같이 SpringResourceTemplateResolver Bean 을 정의해서 templateResolver 정의를 해주면된다 

## application.properties 

```
spring.thymeleaf.prefix=classpath:/templates/view/
spring.thymeleaf.suffix=.html

```

기존설정을 지우고 properties 에 이렇게 정의를 해주면된다 jsp 같은 경우는 boot 로 넘어오면서 권장하는 템플릿 엔진은 아니게 되었고 
spring 이 밀어주는 템플릿 엔진중에 thymeleaf 가 있는것이다 

더 요즘은 자바스크립트 라이브러리를 활용해서 vue , react 를 프론트엔드로 활용을 하고 , spring 은 back-end 전용으로 많이 뺴고 있는 추세이다 

그러면 간단한 form 문을 활용할것이다 하면서 thymeleaf 문법을 보긴 할꺼지만 거진 대부분 바닐라 JS 또는 react 를 활용해서 프로그램을 개발할것이다 
그럼 오늘 주제 ModelAttribute 에 대해서 알아보자 

## hello.html 

```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project 2


<form id="form-table" action="/hello/web" method="get">
    name : <input name = "name">
    age : <input name = "age">

    submit : <input type="submit">

</form>

</body>
</html>

```
이렇게 html 을 수정을 하자 그리고 controller 에 다음과 같이 코딩을 할것이다 

```


@GetMapping("/hello/web")
@ResponseBody
public UserModel UserModel(@ModelAttribute UserModel userModel){
    
    System.out.println("Name : " + userModel.getName());
    System.out.println("Age : " + userModel.getAge());
    
    return userModel;
            
    
}

```
HelloController 이와 같이 핸들러 하나를 추가하고 

UserModel.java

```

package com.cybb.main.model;

public class UserModel {

    private String name;
    private String age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }
}


```

그리고 웹화면에서 값을 넣어보자 본인들 원하는 값을 넣었겠지만 나는 Name : kimdongy1000 Age : 25 넣었다 그리고 화면이 바뀌면서 
아까 우리가 넣었던 값이 key - value 의 Object 값으로 변환되어서 나왔다 일단 웹화면 넘어가는거 까지는 다음에 하고 어떻게 데이터를 받았는지 이다

일단 우리는 핸들러를 다시보자 

```

@GetMapping("/hello/web")
@ResponseBody
public UserModel UserModel(@ModelAttribute UserModel userModel){
    
    System.out.println("Name : " + userModel.getName());
    System.out.println("Age : " + userModel.getAge());
    
    return userModel;
            
    
}

```

우리가 웹화면에서 준 name , age 는 (@ModelAttribute UserModel userModel) 이곳의 UserModel 객체로 받아졌다 그리고 UserModel 은 getName , getAge 가 있는것만 보면
현재 Model 에 어떤 필드가 있는지 알 수 있는것이다 그럼 @ModelAttribute 은 무엇일까?

기본적으로 메서드 매개변수에 바인딩할 모델 속성의 값을 설정하는 데 사용되는 애노테이션입니다.
이 말이 무엇이냐 우리는 앞에서 form 요청으로 name 하고 age 값을 넣고 submit 버튼을 눌렀다 이때 이 요청은 http QueryString 형태로 들어가게 되는데 다음과 같을것이다 

localhost:8080/hello/web?name=kimdongy1000&age=25 이렇게 요청정보가 들어갈것이다 그럼 @ModelAttribute 는 이때 name 하고 age 를 받아서 바로 객체 형태로 부를 수 있게 만든다 그것이 바로 userModel.getName 이다 사실 생각하면 UserModel userModel = new UserModel() 을 하지 않으면 할 수 없는 행동을 하기 때문에 @ModelAttribute 은 
자체 객체를 만들어서 바로호출할 수 있게 만들어주는 것이다 

그리고 핸들러를 한번더 수정을 하겠다 

```

@GetMapping("/hello/web")
public String UserModel(@ModelAttribute("model") UserModel userModel){
    return "hello2";
}

```

```

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project 2 - 핸들러에서 넘어오는 값

name : <input th:value="${model.name}">
age  : <input th:value="${model.age}">




</body>
</html>

```

그리고 hello2 html 을 만들어보자 참고로 th 는 타임리프 문법으로 서버에서 model 로 던져주는 데이터를 th: 문법을 이용해서 데이터를 템플릿엔진으로 바인딩 할 수 있다 
`<html xmlns:th="http://www.thymeleaf.org">` ,  `<input th:value="${model.name}">` 이렇게 말이다 

그럼 여기서 문제 우리는 앞에서 통상적으로 웹페이지에 데이터 바인딩 하기 위해서는 model 이라는 객체를 사용해야 했다 하지만 지금은 model 이라는 값을 넣지 않고 
ModelAttribute 을 통해서 client -> server -> cliet 로 넘기는 것을 보았다 있데 ModelAttribute 은 굳이 데이터 변경건이 없을때에는 바로 Model 객체를 심지 않고도 
clinet 에 데이터를 반환할 수 있는것을 보았다


우리는 여기 까지 @ModelAttribute 가 기본적으로 HTTPS 에서 어떤 동작을 하는지 보았다 기본적으로 mvc 에서 넘어오는 값들을 바로 객체화 해서 사용할 수 있는점과 
@ModelAttribute 선언되어 있으면 굳이 model 에 데이터를 심지 않아도 바로 client 에 보내는거 까지 보았다 마지막으로 

@ControllerAdvice 를 활용해서 전역적으로 사용하는 Model 에 대해서 알아보겠습니다 


GlobalControllerAdvice.java
```

package com.cybb.main.controller.advice;

import com.cybb.main.model.AdminUser;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ModelAttribute;

@ControllerAdvice
public class GlobalControllerAdvice {

    @ModelAttribute("appName")
    public String appName() {
        return "My Application";
    }

    @ModelAttribute("adminUser")
    public AdminUser currentUser() {
        AdminUser user = new AdminUser();
        user.setName("admin");
        user.setEmail("admin.com");
        return user;
    }
}


```

@ControllerAdvice 등록된 데이터는 Controller 에서 전역적으로 사용할 수 있는 무엇인가를 정의할 수 있다 우리는  그 중에서 ModelAttribute 를 메서드 상단에 정의하고 
이 controller 을 전역적으로 사용해서 데이터를 바인딩해서 client 로 넘기는것을 보일것이다 

```

@GetMapping("/hello/web")
public String UserModel(@ModelAttribute("model") UserModel userModel  , @ModelAttribute("adminUser") AdminUser adminUser , @ModelAttribute("appName") String appName){
    return "hello2";
}

```

그리고 위의 핸들러를 지금과 같이 사용합니다 그러면 이 핸들러는 GlobalControllerAdvice 에 있는 @ModelAttribute key 값을 읽어서 화면으로 던져줄것입니다 
그럼 html 은 다음과 같이 변경이 될것입니다 

```

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project 2 - 핸들러에서 넘어오는 값

name : <input th:value="${model.name}">
age  : <input th:value="${model.age}">

adminUser  : <input th:value="${adminUser.name}">
adminEmail  : <input th:value="${adminUser.email}">
appName :  <input th:value="${appName}">




</body>
</html>

```

이렇게 되면 우리는 @ModelAttribute 에 대해서 모두 공부한거 같다 














