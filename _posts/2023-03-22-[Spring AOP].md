---
title: Spring AOP
author: kimdongy1000
date: 2023-03-22 09:07
categories: [Back-end, Spring - Core]
tags: [ AOP ]
math: true
mermaid: true
---

## Aspect-Oriented Programming 
일명 OP 는 프로그램 구조에 대한 또 다른 사고 방식을 제공하여 객체 지향 프로그래밍 OOP 을 보안합니다 
OOP 에서 모듈화의 핵심 단위는 클래스인 반면 AOP 에서는 모듈화 단위가 Aspect AOP 는 IoC 컨테이너에 의존되지 않은 독립적 프로그램입니다 

## AOP 의 개념 

 1. Aspect : 핵심기능 코드 사이에 침투한 부가기능을 모아놓은 모듈이다 + 어디에 적용시킬것인지에 대한 적용 (포인트컷)
             예를 들어서 A 라는 테이블에 값을 집어 넣는다고 했을때 
                
                a. 트랜잭션 시작 
                b. AutoCommit false 
                c. 핵심기능 (데이터 삽입)            
                d. AutoCommit true 
                e. commit (트랜잭션 종료)
이때 부가기능을 모아놓은 모듈을  Advice 라고 하고 어디에 적용할지를 PointCut 이라고 한다 이이때 Aspect 는 이 Advice + PointCut 을 합친거라고 생각하면 된다 

2. PointCut : Advice 가 실행될 위치 

3. Advice : 부가기능을 모아놓은 모듈

## Aspect 지원 
Aspect 는 일반적으로 java code 로 선언하는 방식이 있고 전통적인 xml 방식으로 정의를 해두기도 합니다 우리는 java 코드로만 정의를 해서 사용을 해보겠습니다 



AOP 의존성 추가
```

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

```


AppConfig
```

@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}

```

`@EnableAspectJAutoProxy` 를 통해 Aop 를 활성화 할 수 있습니다 


LoggerAspect.java
```

package com.cybb.main.aop;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class LoggerAspect {

    @Pointcut("execution(* com.cybb.main.controller..*.*(..))")
    private void logPointCut(){}

    @Before("logPointCut()")
    public void before(JoinPoint joinPoint){

        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method =  methodSignature.getMethod();
        System.out.println(method.getName() + "실행");


    }


}






```
@Aspect 애노테이션을 달아서 이 클래스에서 Aspect 정의를 하겠다는 뜻입니다 이때 이 클래스는 메서드와 필드를 가질 수 있고 pointCut , Advice 를 선언할 수 있습니다 
그리고 Bean 에 감지가 되어야 함으로 @Component 를 사용하여 줍니다 Spring 공식문서에는 단순 @Aspect 만으로 Bean 을 감지할 수 없기에 이 Bean 을 감지할 수 있는 애노테이션 
또는 Bean 을 선언을 해주어야 한다고 적혀 있습니다 (싱글톤 구성)


## PointCut 선언 
```
@Pointcut("execution(* com.cybb.main.controller..*.*(..))")
private void logPointCut(){}

```
포인트컷은 이렇게 선언할 수 있는데 위에 `@Pointcut("execution(* com.cybb.main.controller..*.*(..))")` 어디서 실행할것인지 정의한것이고  `private void logPointCut(){}` 부분은 실제 위치에서 사용할 메서드인것입니다 

그럼 포인트컷의 표현식에 대해서 알아야 하는데 

1. execution 메서드 실행 조인 포인트를 매핑한다 스프링 AOP 에서 가장 많이 사용되는 표현식중에 하나이다 
    a. 이때 "execution(* com.cybb.main.controller..*.*(..))" 있는 것들은 다음의 패턴을 따르는데 
        i. 첫번째 * 는 접근제한자 패턴이다 와일드 카드를 쓰면 접근제한자 상관없이 적용하겠다는 뜻이고 
            a. "execution(private com.cybb.main.controller..*.*(..))"  라고 사용하면 private 접근제한자를 가진 메서드만 Aop 를 사용하겠다는 뜻이다 
        ii. 두번째 패키지와 클래스 이름에 대한 패턴 생략이 가능하며 연결할때는 . 을 사용함 그래서 지금 이  com.cybb.main.controller..*.* com.cybb.main.controller 하위에 있는 모든 패키지에 대해서 모든 클래스의 모든 메서드를 뜻한다 
        iii. 그리고 마지막 (..) 은 마지막 메서드에 붙어 나오는 것으로 파라미터 패턴을 넣을 수 있다 (..) 지금 포현은 어떤 파라미터 상관없고 개수 상관 없는 것을 말한다 

이 execution 표현식이 가장 많이 쓰이는 방식중에 하나이다         

## Advice 선언 

```
@Before("logPointCut()")
public void before(JoinPoint joinPoint){
    
    MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
    Method method =  methodSignature.getMethod();
    System.out.println(method.getName() + "실행");
    
    
}

```

이렇게 실행할 수 있다 이때 우리는 @Before 메서드가 실행되기 이전에 실행 이때 이전이라고 하면 이해가 안되지만 난중 다음 실제 호출했을때 어떻게 적용되는지 살펴보면 알게 된다 

## 핵심 코드 작성 

```

package com.cybb.main.controller;

import com.cybb.main.domain.User;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HomeController {

    @PostMapping("/insertUser")
    public User insertUser(
            @RequestParam("name") String name ,
            @RequestParam("age") int age
        ) throws Exception
    {
        try{

            User user = new User();

            System.out.println(name);
            System.out.println(age);

            user.setName(name);
            user.setAge(age);
            
                    
            return user;

        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}


```

이렇게 이제 핸들러 하나를 만들어보자 핵심로직에서는 아무런 작업을 할 필요 없다 우리가 위에서 선언한 Aop 규칙으로 Srping 이 알아서 Advice 를 원하는 위치에 넣게 된다 
그렇게 실행을 하고 핸들러를 호출하면 

```
insertUser실행
kimdong
10

```

우리가 선언하지 않음 System.out.println 가 나오게 된다 그럼 이제 우리는 앞에서 메서드 실행 이전에 대한 기준을 알게 된것이다 여기서 이전은 메서드 (여기선 핸들러) 가 호출되고 나자마자 호출되는 것을 @Before 로 선언하는 것이다 마찬가지로 @After 이 있는데 마저 작성하면

## @After
```

@After("logPointCut()")
public void after(JoinPoint joinPoint ){

    MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
    Method method =  methodSignature.getMethod();
    System.out.println(method.getName() + "종료");
    

}

```

그럼 전체 결과를 보자 

```

insertUser실행
kimdong
10
insertUser종료

```

그러면 실행과 종료를 이렇게 찍었다 다만 여기서는 After 의 좀더 다른 기능을 사용해보자 

After 은 AfterReturing ,  AfterThrowing 가 존재하는데 성공적으로 마쳤을때 Afterreturing 를 사용해보자 

## @AfterReturning
```

@AfterReturning(value = "logPointCut()" , returning = "object" )
public void after(JoinPoint joinPoint , Object object){
    System.out.println("return Object" + object);


}

```

이는 핵심로직에서 return 데이터가 있을때 사용할 수 있는 pointcut 이며 위의 After 와 같이 사용하면 위와 같은 결과가 나온다 

```

insertUser실행
kimdong
10
return Objectcom.cybb.main.domain.User@7ca2a82e
insertUserafter 종료

```

즉 @AfterReturning 먼저 호출이 되고 @After 가 호출되는것을 볼 수 있다 


마지막 에러 발생했을때 후처리 하는 AfterThrowing 를 찾아보자 
그럼 핸들러를 약간 수정해서 

```

package com.cybb.main.controller;

import com.cybb.main.domain.User;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HomeController {

    @PostMapping("/insertUser")
    public User insertUser(
            @RequestParam("name") String name ,
            @RequestParam("age") int age
        ) throws Exception
    {
        try{

            User user = new User();
            
            if(!StringUtils.hasText(name)){
                throw new RuntimeException("name 이 올바르지 않습니다");
            }

            System.out.println(name);
            System.out.println(age);


            user.setName(name);
            user.setAge(age);



            return user;

        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}


```

파라미터를 검증해서 문제가 생기게끔 만들어보자 

## @AfterThrowing
```

@AfterThrowing(value = "logPointCut()" , throwing = "ex")
public void afterThrowing (JoinPoint joinPoint , Throwable ex){
    
    System.out.println("===========Exception 발생==============");
    System.out.println("ex.getCause() : " + ex.getCause());
    System.out.println("ex.getMessage() : "  + ex.getMessage());
    System.out.println("ex.getStackTrace() : " + ex.getStackTrace());
    
}

```

자그럼 우리는 요청을 할때 핸들러 파라미터를 이상하게 보낼것이다 지금은 name 을 비어 있는 값을 보내면된다 

```

insertUser실행
===========Exception 발생==============
ex.getCause() : java.lang.RuntimeException: name 이 올바르지 않습니다
ex.getMessage() : java.lang.RuntimeException: name 이 올바르지 않습니다
ex.getStackTrace() : [Ljava.lang.StackTraceElement;@1d40326f
insertUserafter 종료
2023-03-22 11:00:23.436 ERROR 13220 --- [nio-8080-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: java.lang.RuntimeException: name 이 올바르지 않습니다] with root cause

java.lang.RuntimeException: name 이 올바르지 않습니다

```

이걸게 결과가 나온다 먼저 @Before 가 실행이 된다 그러다 에러 발생해서 @AfterThrowing 에 ㅇㅆ는 메서드를 호출 그리고 @After 는 오류 유무와 상관 없이 항상 호출이 됩니다 
그리고 그 이후 나오는 메세지는 실제 핸들러의 마지막 catch 부분에서 동작하는 thro new RuntimeException 에 동작되는 메세지입니다 

우리는 그러면 Aop 의 처음과 끝까지 한번 코딩을 해보았고 다음 포스터에서 간단한 프로그램을 한번 만들어보겠습니다 
























