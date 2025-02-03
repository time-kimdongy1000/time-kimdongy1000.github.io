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
소프트웨어에 개발에서 코드를 모듈화하는 방법 중에 하나로 관심사를 중심으로 코드를 구조화하는 기법입니다 횡단 관심사와, 핵심 관심사를 모듈화해서 분리하고
횡단 관심사를 여러 모듈 코드에서 중복 없이 사용할 수 있도록 도와줍니다

## 횡단 관심사 , 핵심 관심사 
예를 들어서 다음의 로직이 있다 이 중에서 횡단 관심사와, 핵심 관심사를 분리해 보자

```
로직 1.
StartTranscation()
SetAutoCommit(false)
insertMember(user)
commit()
setAutoCommit(true)

로직 2.
StartTranscation()
SetAutoCommit(false)
updateMember(user)
rollback();
setAutoCommit(true)
```

이 두 개의 로직 중에 횡단 관심사와, 핵심 관심사를 분류를 해보면 핵심 관심사는 결국 비즈니스 로직일 수밖에 없다 그렇기 때문에 새로운 멤버를 추가하는 inserMember이나 기존의 멤버를 업데이트하는 updateMember 가 핵심 관심사이고 그 사이에 트랜잭션에 관련한 모든 것들을 횡단 관심사라고 한다 즉 Aop 이런 횡단 관심사들이 비즈니스 로직이 반복됨에 따라
계속해서 동일하게 반복되고 있는 것을 볼 수 있는데 AOP는 이런 횡단 관심사를 한 곳에 묶어서 중복으로 호출하지 않게 해서 개발자는 비즈니스 로직에만 신경 쓰는 프로그래밍 기법을 말합니다

## AOP 의 용어 

1. Aspect  
    AOP에서의 모듈화 단위. 횡단 관심사를 담당하는 코드의 집합. Aspect는 어떤 특정한 관심사를 구현한 코드 모듈을 나타냅니다.

2. Advice 
    횡단 관심사를 구현한 코드. 메소드 실행 전, 후 또는 예외 발생 시 등과 같이 핵심 관심사에 결합될 수 있습니다.

3. Join Point
    Advice가 핵심 관심사에 결합되는 지점. 메소드 호출, 객체 생성 등과 같은 프로그램 실행 중의 특정 시점입니다.

4. Pointcut 
    어떤 Join Point에서 Advice를 실행할 것인지를 결정하는 표현식. 특정 메소드 호출, 특정 패키지 내의 모든 메소드 등과 같이 지정할 수 있습니다.


## AOP 의 장점은
1. 모듈화 
    관심사를 분리함으로서 코드의 모듈화를 증가시킵니다 

2. 재사용성
    동일한 관심사를 여러 모듈에서 중복으로 구현하지 않고 재사용할 수 있습니다 

3. 유지보수성 
    핵심 관심사와 횡단 관심사가 분리되어서 코드를 더 쉽게 이해하고 유지보수 할 수 있습니다 


## AOP 단점 
1. 복잡성과 어려운 디버깅 
    AOP 를 사용하면 코드의 흐름이 분산되기 때문에 디버깅 및 코드의 이해가 어려워질 수 있습니다 어떤 횡단 관심사가 언제 실행되는지 파악하기 어려운것이 단점입니다 

2. 성능 오버헤드
    AOP 가 코드를 실행중에 추가적인 동작을 수행함으로 성능 오버헤드가 발생할 수 있습니다 

3. 횡단 관심사의 재사용의 어려움
    특정 모듈에 횡단 관심사가 강력하게 결합을 하면 다른 모듈에서 사용하기 어려운 점이 있습니다 



## 간단한 AOP 프로젝트 만들기 

우리가 할 간단한 프로그램은 aop를 활용해서 autoCommit 을 푸는 거부터 해서 insert 가 제대로 되면 commit 안되면 rollback 하는 횡단 관심사를 만들어서 진행을 하겠습니다
필요한 maven 은 web , mybatis , ojdbc , aop를 사용하겠습니다

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


<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>

<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>21.6.0.0.1</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

```

## AopConfig
```
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {

}


```

먼저 Aop를 사용할 conifg를 작성합니다 이 config는 aop를 사용할 때 쓰는 설정 파일이지만 지금은 그냥 bean만 생성할 수 있게 하겠습니다 그리고 `@EnableAspectJAutoProxy`는
AspectJ 기반의 AOP를 사용하기 위해 자동 프락시 생성을 활성화하는 애노테이션입니다


## 횡단 관심사 만들기 
```
@Aspect
@Component
public class DbAspect {

    private SqlSession sqlSession;
    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;


    @Before("execution(* com.cybb.main.service..*.*(..))")
    public void transactionStart(JoinPoint joinPoint) throws Exception{
        System.out.println("===== 트랜잭션 시작 =====");

        sqlSession = sqlSessionTemplate.getSqlSessionFactory().openSession();
        Connection connection = sqlSession.getConnection();
        connection.setAutoCommit(false);
    }

    @AfterReturning(pointcut = "execution(* com.cybb.main.service..*.*(..))" )
    public void transactionEndCommit(JoinPoint joinPoint) throws Exception{
        System.out.println("===== 트랜잭션 종료 Commit =====");

        Connection connection = sqlSession.getConnection();

        connection.commit();
        connection.setAutoCommit(true);

        sqlSession.close();
    }

    @AfterThrowing(pointcut = "execution(* com.cybb.main.service..*.*(..))" , throwing = "ex")
    public void transactionEndRollBack(JoinPoint joinPoint) throws Exception{
        System.out.println("===== 트랜잭션 종료 에러발생 Rollback =====");

        Connection connection = sqlSession.getConnection();

        connection.rollback();
        connection.setAutoCommit(true);

        sqlSession.close();
    }

    @After("execution(* com.cybb.main.service..*.*(..))" )
    public void transactionEnd(JoinPoint joinPoint) throws Exception{
        System.out.println("===== 로직 종료 =====");

    }
}


```
이 부분이 이제 횡단 관심사이다 이 횡단 관심사는 우리가 지정한 pointCut에 들어가서 자동으로 붙게 된다 여기서 하나하나 알아보자

1. @Before 
    지정한 포인트컷에서 제일 먼저 실행되는 횡단 관심사입니다 

2. @After 
    지정한 포인트컷에서 제일 마지막에 실행됩니다 이때는 핸들러의 에러여부 상관 없이 무조건 실행됩니다 

3. @AfterThrowing
    지정한 포인트컷에서 에러가 발생했을때 동작하는 횡단 관심사입니다

4. @AfterReturning
    지정한 포인트컷에서 에러없이 끝이 났으면 실행되는 횡단 관심사입니다

그렇다는 플로우는 이렇게 된다 

1. @Before -> 에러 없이 동작 완료 -> @AfterReturning -> @After

2. @Before -> 에러 발생 -> @AfterThrowing -> @After

이렇게 두가지 플로우를 따르게 됩니다 

## PointCut 
우리는 위에서 계속 지정한 포인트 컷이라고 했다 이 뜻이 무엇이냐면 포인트 컷은 이 횡단 관심사가 실행될 위치를 명시할 수 있습니다
`pointcut = "execution(* com.cybb.main.service..*.*(..))"` 이렇게 선언된 것들이 pointcut입니다 이렇게 명시적으로 사용할 수 있습니다 그럼 이 뜻이 무엇이냐

1. execution 
    메서드 실행 조인 포인트를 매핑합니다 스프링 AOP 에서 가장 많이 사용하는 표현식중 하나입니다 즉 횡단 관심사를 어떤 메서드에 매핑을 시키기 위한 표현식이라고 생각하시면됩니다 
    
    execution(modifiers-pattern? return-type-pattern declaring-type-pattern? method-name-pattern(param-pattern) throws-pattern?)

    표현식의 구조는 이렇게 되는데 
        a. modifiers-pattern 메서드의 접근제한자를 나타내며 생략이 가능합니다 
        b. return-type-pattern 메서드의 반환타입을 나타내며 생략이 가능합니다 
        c. declaring-type-pattern 메서드를 선언한 클래스나 인터페이스 패턴을 나타내며 생략이 가능합니다 
        d. method-name-pattern 메서드의 이름을 나타냅니다 
        e. param-pattern 매서드의 매개변수를 나타냅니다 
        f. throws-pattern 메서드에서 던지는 예외를 나타냅니다 

2. execution(* com.cybb.main.service..*.*(..)) 분리 
    a. * 를 접근제한자 모든 접근제한자에서 사용하겠다는 뜻입니다         
    b. com.cybb.main.service 이 패키지 아래에 있는 곳에서 사용을 하겠다는 뜻입니다 이때 .. 은 와일드 카드로 이 service 하단에 있는 모든 패키지를 포함하게 됩니다 
    c. *.* 이는 모든 클래스.모든 메서드 를 나타내며 
    d. (..) 모든 매개변수를 뜻합니다

종합하면 이 포인트 컷은 com.cybb.main.service 포함 및 하단에 있는 모든 패키지에 이 포인트 컷을 적용하겠다는 뜻입니다

## 핵심 관심사 적용
```
package com.cybb.main.service;

@Service
public class MemberService {

    @Autowired
    private MemberRepository memberRepository;

    public void insertMember(){

        Map<String , Object> dbParam = new HashMap<>();
        dbParam.put("name" , "Time");
        dbParam.put("age" , 20);

        memberRepository.insertMember(dbParam);
    }
}

```
원래는 패키지 명을 안 쓰는데 이번에는 필요할 거 같아서 패키 지명을 남깁니다 지금 보면 핵심 관심사는 비즈니스 로직으로 새로운 member를 넣는 작업을 하게 됩니다 이를 실행하게 되면
Spring AOP는 알아서 @Before부터 @After까지 전부 실행을 하게 됩니다 그럼 실행을 해보자

## AfterReturing 
```
===== 트랜잭션 시작 =====
===== 트랜잭션 종료 Commit =====
===== 로직 종료 =====

```
로직 시작과 종료가 올바르게 끝이 나서 @AfterReturning -> @After로 간 것을 볼 수 있습니다 그렇다면 일부로 오류를 발생시키면

## AfterThrowing 
```

===== 트랜잭션 시작 =====
===== 트랜잭션 종료 에러발생 Rollback =====
===== 로직 종료 =====

```
일부로 에러를 발생시키면 이렇게 @AfterThrowing -> @After로 가게 됩니다 오늘은 Aop를 활용해서 개발자는 비즈니스 로직에만 중점을 개발을 하고 횡단 관심사는 분리를 해서
다른 곳에서 개발한 후 적용을 하는 Aop에 대해서 알아보았습니다