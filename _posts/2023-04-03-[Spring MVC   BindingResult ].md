---
title: Spring MVC BindingResult
author: kimdongy1000
date: 2023-04-03 09:37
categories: [Back-end, Spring - MVC]
tags: [ MVC , BindingResult ]
math: true
mermaid: true
---

## Spring MVC 유효성 검사
우리는 보통 웹개발할때 유효성 검사를 하는 편이다 단순한 요청이면 프런트에서 , 보안이 예상된다면 프런트 , 백단에서 둘다 유효성 검사를 해야 한다 
이번에는 프런트에서 유효성 검사하는 방법하고 백단에서 유효성 검사를 하는 방법에 대해서 공부를 해볼려고 한다 

이번에는 이름 , 나이 , email 을 가지고 각각 유효성 검사를 진행을 하겠다 그전에 어떻게 제약조건을 걸지는 다음과 같이 정한다 

성과 이름은 알파벳 영어로 구성되며 이름 성 순으로 표기하되 반드시 이름 하고 성 사이엔 띄어쓰기가 되어 있어야 하며 하나의 필드로 저장된다 
나이는 숫자마 가능하며 0~ 100 사이로만 표기 가능하다 


그럼 간단한 form 하고 js 를 만들어보자 이때 spring boot 는 어떻게 js 파일을 가지고 와서 읽는지 살펴보자 

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
    name : <input class = "firstName" name = "firstName" placeholder="firstName"> <input  class = "secondName" name = "secondName" placeholder="secondName">        </p>
    age : <input  class = "age" name = "age">                                                                                                                       </p>
    

    submit : <input class = "form-button" type="button" value = "제출">

    

</form>

<script src = "/js/hello.js"></script>
</body>
</html>

```

form 을 살펴보기 전에 spring boot 의 정적 리소스 관리에 대해서 조금만 공부를 하고 가자 spring boot 는 기본적으로 정적 리소스 설정을 자동으로 해준다 
이는 기존 spring 하고는 다르게 static 아래에 있으면 boot 는 이 아래에 있는 모든 자원을 정적 자원관리로 등록을 하고 어디에서든 불러서 사용할 수 있다 

![1](https://user-images.githubusercontent.com/58513678/229422993-b8524224-ae64-4e31-8a15-89c6a46f96ad.jpg)



기본적으로 현재 보이는 hello.js 파일위치를 보면 이것이 boot 에서 선언하는 기본위치이다 이 위치를 바로 사용할 수 있지만 다른 path 를 자원관리 위치로 설정을 할때는 어떻게 해야 할지 살펴보자 

```

package com.cybb.main.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/js/**").addResourceLocations("classpath:/static/js/");
    }
}

```

이렇게 @Configuration addResourceHandlers 를 지정하면 /js 아래로 요청이 들어오는 모든건들은  resource 를 classpath:/static/js/ 여기 아래에서 찾으세요 
Spring boot 는 이 설정을 안해도 되지만 spring 은 이 설정을 해주어야 resource handler 를 읽어낼것이다 

그럼 다른위치도 동작시킬 수 있는지 살펴보자 

![1](https://user-images.githubusercontent.com/58513678/229426414-14beac02-d657-4f4b-8747-4f986222cf26.jpg)



우리는 다른 위치에 hello2.js 파일을 만들고 한번 html 에서 불러보자

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
    name : <input class = "firstName" name = "firstName" placeholder="firstName"> <input  class = "secondName" name = "secondName" placeholder="secondName">        </p>
    age : <input  class = "age" name = "age">                                                                                                                       </p>
    

    submit : <input class = "form-button" type="button" value = "제출">

</form>

<script src = "/js/hello.js"></script>
<script src = "/js/customJs/js/hello2.js"></script>
</body>
</html>

```

`<script src = "/js/customJs/js/hello2.js"></script>` 추가 되었다고 할지라도 실행을 하면 이 js 파일을 읽을 수 없을것이다 그 이유는 간단하다 boot 가 설정한 기본위치 말고는 읽을 수 없기 때문이다 

그럼 저 위치를 addResourceHandlers 매핑을 시켜주자 

```
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/js/**").addResourceLocations("classpath:/static/js/" , "classpath:/js/customJs/");
}

```

그리고 html 또한 수정을 해주자 

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
    
    name : <input class = "firstName" name = "firstName" placeholder="firstName"> <input  class = "secondName" name = "secondName" placeholder="secondName">        </p>
    age : <input  class = "age" name = "age">                                                                                                                       </p>
    

    submit : <input class = "form-button" type="button" value = "제출">

</form>

<script src = "/js/hello.js"></script>
<script src = "/js/hello2.js"></script>
</body>
</html>

```

자 우리가 얼핏보면  `<script src = "/js/hello.js"></script>` classpath 아래에 있는 정적 리소스 자원을 절대위치로 찾아오는것 처럼 보이지만 실제로는 모두 http 통신을 하는것이다 get 요청으로 localhost:8080/js/ 라고 요청을 넣으면 `<script src = "/[매핑주소]/[찾을 파일명]"></script>` 이렇게 되는것이다 
그럼 이제 spring 은 classpath:/static/js/ 위치와 classpath:/js/customJs/ 아래 위치에서 hello.js 그리고 hello2.js 파일을 찾게 된다 그리고 파일이 있으면 js 정적파일을 등록하고 스크립트 파일로 읽을 수 있게끔 만들어주는것이다 

그럼 간단하게 spirng 이 어떻게 정적자원을 관리하는지 보았다 

그럼 우리는 기본정책을 사용해서 계속해서 JS 로 먼저 유효성 검사를 진행을 해보자 그럼 여기서는 ES6 자바스크립트 문법을 활용해서 프런트 유효성검사를 진행을 할것이다 


## hello.html

그럼 다시 html 을 적용해보자 

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
    name : <input class = "firstName" name = "firstName" placeholder="firstName"> <input  class = "secondName" name = "secondName" placeholder="secondName">        </p>
    age : <input  class = "age" name = "age">                                                                                                                       </p>


    submit : <input class = "form-button" type="button" value = "제출">

</form>

<script src = "/js/hello.js"></script>

</body>
</html>

```

이제 우리는 hello.js 로 제약조건을 검색해서 진행을 해보자 

## 프런트 검증
```

// DOM 객체 찾아오기
const firstName = document.querySelector("#form-table .firstName");
const secondName = document.querySelector("#form-table .secondName");
const age = document.querySelector("#form-table .age");
const form_button = document.querySelector("#form-table .form-button");
const form_table = document.querySelector("#form-table");


// Click 이벤트 생성
form_button.addEventListener('click' , (e => {

    const firstName_value = firstName.value;
    const secondName_value = secondName.value;
    const age_value = age.value;


    const rex_name = new RegExp('[a-zA-Z]');


    if(!firstName_value.match(rex_name) || !secondName_value.match(rex_name)){
        alert('성 또는 이름 형식이 올바르지 않습니다.');
        return
    }

    if(age_value >= 0 && age_value <= 100){
        alert('나이 형식이 올바르지 않습니다.');
        return
    }
    
    form_table.submit();

}))


```
간단하게 제약조건에 대해서 정규식으로 파악하고 정규식은 다음에 공부할 일이 있을테니 일단 지금은 정규식이라고 특정 문자열 패턴으로 현재 문자열이 올바른 패턴인지 아닌지 판단하는 라이브러리이다 
프로그래밍언어 거의 대부분 같은 패턴으로 걸러낼 수 있다 `const rex_name = new RegExp('[a-zA-Z]');` 간단하게 적으면 rex_name 은 패턴을 검색할때 a-zA-Z 즉 숫자 없이 a-zA-Z 사이에 있는 알파벳만 
사용해서 검사를 하겠다는 것이다 이때는 알파벳 외에는 전부 패턴에서 false 를 반환한다 

그럼 여기서는 프런트에서 원하는 제약조건으로 잘 걸러지게 된다 하지만 프런트는 언제든지 데이터를 조작할 수 있게 된다 즉 이 웹페이지를 통하지 않더라도 데이터를 충분히 보낼 수 있다 
postman 이나 또는 단순 스크립트 조작으로 제약조건을 태우지 않고 보낼 수 있게 된다 그래서 서버에서 한번더 이 데이터를 검사할것이다 

## 서버에서 검증 
혹시 우리 앞에서 spring - core 에서 배운것이 기억이 날까 앞에서 우리는 Validation 에 대해서 공부를 했다 spring 고유 기능으로 첫번째 서버 검증은 이 기능을 활용해서 진행을 할것이다 


HelloController.java

```

package com.cybb.main.controller;

import com.cybb.main.model.UserModel;
import com.cybb.main.model.valid.UserModelValid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.validation.BeanPropertyBindingResult;
import org.springframework.validation.Errors;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.ResponseBody;


@Controller
public class HelloController {

    @Autowired
    private UserModelValid userModelValid;

    @GetMapping("/hello")
    public String helloWorld(){

        return "hello";
    }

    @GetMapping("/hello/web")
    @ResponseBody
    public String userModel(@ModelAttribute UserModel userModel){

        try{

            Errors errors = new BeanPropertyBindingResult(userModel , "userModel");
            userModelValid.validate(userModel , errors);

            errors.getAllErrors().forEach(e -> {
                throw new RuntimeException(e.getDefaultMessage());
            });

            return userModel.toString();

        }catch(Exception e){
            return e.toString();

        }
    }
}


```

UserModelValid.java
```
package com.cybb.main.model.valid;

import com.cybb.main.model.UserModel;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;

import java.util.regex.Pattern;

@Component
@RequestScope
public class UserModelValid implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return UserModel.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {

        UserModel userModel = (UserModel)target;

        String nameRex = "^[a-zA-Z]*$";

        if(!userModel.getFirstName().matches(nameRex) ){
            errors.rejectValue("firstName" , "first name inValid" , "이름의 유효성이 올바르지 않습니다.");

        }

        if( !userModel.getSecondName().matches(nameRex)){
            errors.rejectValue("secondName" , "second name inValid" , "이름의 유효성이 올바르지 않습니다.");
        }

        if(userModel.getAge() < 0 || userModel.getAge() > 100){
            errors.rejectValue("age" , "age inValid" , "나이 유효성이 올바르지 않습니다.");
        }

    }
}


```

우리는 core 단에서 Validator 사용했다 그것을 이용해서 controller @ModelAttribute UserModel userModel 이것으로 받으면 바로 검증을 해서 진행을 할것이다 이때 
달라진것은 우리는 앞에서 new 를 사용해서 UserModelValid 객체를 만들어 냈지만 사실 UserModelValid 는 메모리 하나만 해서 여러곳에서 공통으로 사용해도 상관이 없다 다만 
지연이 발생할때에는 다른 메모리에 겹쳐쓸 확률이 있음으로 이때는 요청할때마다 새로운 객체를 만들어내는것이 맞기 때문에 @RequestScope 로 요청이 들어올떄마다 만들어내고 http 요청이 끝이나면 자동으로 소멸되는 이를 사용해서 관리할것이다 그럼 오류가 발생할때에는 웹페이지에 오류내용이 표기 될것이다 

물론 이렇게 개발을 해도 진행을 해도 된다 보통 form 문으로 돌아가거나 아니면 통합 오류 페이지를 만들어서 redirect 시키는게 일반적이지만 이번에는 다른 방법으로 오류 검증을 하고
그 검증에 오류가 발생하면 다시 오류 내역하고 form 문으로 돌아가는방식으로 개발을 진행을 해보자 

## BindingResult 이것을 활용해서 이번엔 적용을 해보자 

```

<!DOCTYPE html>
<html lang="en">
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project 2


<form id="form-table" action="/hello/web2" method="get" th:object="${user}">
    name : <input class = "firstName" name = "firstName" placeholder="firstName"> <input  class = "secondName" name = "secondName" placeholder="secondName">        </p>

    <p th:if="${#fields.hasErrors('firstName')}" th:errors="*{firstName}">이름의 형식이 잘못되었습니다.</p>
    <p th:if="${#fields.hasErrors('secondName')}" th:errors="*{secondName}">이름의 형식이 잘못되었습니다.</p>

    age : <input  class = "age" name = "age">                                                                                                                       </p>
    <p th:if="${#fields.hasErrors('age')}" th:errors="*{age}">나이의 형식이 잘못되었습니다.</p>


    submit : <input class = "form-button" type="button" value = "제출">

</form>

<script src = "/js/hello.js"></script>

</body>
</html>

```
html 을 수정하자 th 는 타임리프 문법으로 th:if 는 해당 필드의 에러내역이 존재하면 이 `<p>` 를 띄울것이다 그리고 핸들러 하나를 더 만들어보자


```

@GetMapping("/hello/web2")
public String userModel2(@ModelAttribute("user") UserModel userModel , BindingResult bindingResult){

    try{

        userModelValid.validate(userModel , bindingResult);

        if(bindingResult.hasErrors()){
            return "hello";
        }


        return userModel.toString();

    }catch(Exception e){
        throw new RuntimeException(e);

    }
}

```
기존 핸들러와 차이를 보여줘야 하는데 이때 사용하는것이 BindingResult bindingResult 이다 그럼 필드에 에러가 발생하면 자동으로 if 문으로 빠지게 되고 이때 다시 form 문으로 돌아와서
오류가 있는 문구가 나오게 되는것이다 

우리는 이렇게 web 에서 넘어오는 파라미터를 controller 에서 프런트에서 진행한것을 보았고 그리고 앞에서 배운 Validator 로 구현체를 만들어서 유효성 검사를 진행했다 
그 결과를 통합 오류 페이지로 redirect 로 전달하거나 , 또는 다시 form 문으로 전달하는것을 보았다 오류 메세지와 함께 말이다 



































