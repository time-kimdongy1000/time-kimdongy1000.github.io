---
title: Spring Ioc 와 DI 
author: kimdongy1000
date: 2023-03-09 20:17
categories: [Back-end, Spring - Core]
tags: [IoC]
math: true
mermaid: true
---

## IoC 컨테이너
IoC inversion of control 제어의 반전 이때 제어는 객체의 생성 주도권을 의미하는데 이전에 우리가 어떤 객체를 만들었는지 생각을 해보자 

```
Student student = new Student();
```

우리가 직접 main 메서드 안에 new 라는 객체 생성 연산자를 이용해서 생성자를 호출하여 객체를 만들었다 하지만 이제 이런 작업들은 모조리 프레임워크단에서 컨트롤을 할것인데
그것이 바로 제어의 반전 이라는 뜻이다 

## Bean
Spring 에서 Ioc 컨테이너에의해 관리되는 객체를 빈(Bean)이라고 합니다 Bean 은 Spring Ioc 컨테이너에 의해 인스턴스화 , 조립 및 관리되는 객체입니다 
예를 들어서 아래와 같은 new 연산자로 쓰인 객체는 Bean 이 될 수 없습니다 그 이유는 new 라는 객체는 Ioc 컨테이너에 의해서 관리되는 객체가 아니기 떄문입니다

```
Student student = new Student();
```

그럼 뭐가 Bean 이 된다는 것일까? 

## ApplicationContext
ApplicationContext 인터페이스 자체는 IoC 컨테이너 자체를 나타내며 Bean 의 인스턴스화 , 조립을 담당합니다 이때 Bean 이 될 수 있는 개체들은 xml , 애노테이션 , java 코드로 표현이 됩니다 ApplicationContext 도 인터페이스 임으로 다른 구현체로 사용을 해야하는데 보통 ClassPathXmlApplicationContext 를 사용하게 됩니다 예를 들어서 bean 의 정의를 bean.xml 파일로 정의를 해두었을때 이를 Ioc 컨테이너가 읽게끔 만들고 싶으면 아래와 같은 코드를 사용하면됩니다 

```

ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");

```



## 전통적인 방식으로 xml Bean 생성 
예전 레거시 소스를 들여다 보면 xml 에 무엇인가 잔뜩 적혀 있는것을 본적이 있을것이다 거의 대부분의 초심자들은 이런 방대한 xml 파일을 보고 spring 에 정떨어지는 경우를 많이 보았지만 우리는 그렇게 하지 않고 간단간단한 파일들을 한번 보는것으로 진행을 해보자 아까도 보았듯이 ClassPathXmlApplicationContext 를 활용해서 Ioc 의 대상이 되는 ApplicationContext 인터페이스에 Bean 을 생성할 수 있는 정의서를 만들고 기동을 할때 컨테이너에게 읽게끔 시키는 코드를 작성할것입니다 


## 개발환경 
```
java 8

spring boot 2.7.1

maven 

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>

```

maven 은 web 만 가져와도 왠만한 spring 관련해서는 다 사용할 수 있다 그럼 다음의 코드를 보자 원래는 spring 레거시에서 진행을 해야 하지만 사실상 spring 레거시 프로젝트를 구하는게 사실상 어렵고 차세대로 boot 를 많이 씀으로 우리는 boot 를 활용해서 진행을 해볼려고 합니다

## bean.xml

```

<?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
	  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	  xmlns:context="http://www.springframework.org/schema/context"
	  xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd">
		
	  	<bean id = "MySystemInfo" class = "com.cybb.main.MySystemInfo"></bean>

  </beans>

```

그리고 이 파일이 핵심이다 물론 요즘은 이런식으로 Bean 설정파일을 만들지는 않지만 간혹 레거시한 프로젝트에서는 이렇게 bean 을 만들고 관리하고 있다 
여기서의 핵심파일이 bean.xml 인데 설명을 곁들이면 

`<bean id = "MySystemInfo" class = "com.cybb.main.MySystemInfo"></bean>` 이 부분이 bean 을 정의하겠다는 명세서입니다 

* 속성은 id 별 정의를 식별하는 문자열로 정의를 하고 
* class 는 해당 bean을 생성하고자 하는 완벽한 위치를 적어주는것입니다 

그럼 현재 MySystemInfo 식별자를 가지고 있는 bean 이 생겨나게 되는데 해당 bean 의 위치는 com.cybb.main.MySystemInfo 게 되는것이다 
그럼 이 bean 명세서를 가지고 우리는 ApplicationContext 에 주입을 해보겠습니다 

## 컨테이너 인스턴스화 
그럼 우리는 위에서 작성한 MySystemInfo 클래스를 이렇게 만들어보도록 하겠습니다 

```
public class MySystemInfo {
	
	public MySystemInfo() {
		System.out.println("MySystemInfo Bean 생성");
	}	
}
```

우리는 간단하게 class 와 생성자를 만들고 생성자 안에 객체 생성이 되었으면 자도응로 호출되는 생성자 안에 System.out.println 을 찍어보겠습니다.
그럼 이제 컨테이너에 외부에서 생성한 메타데이터 (bean.xml) 을 넣는 과정을 보여드리겠습니다 앞에서 말씀드렸다 싶히 저는 SpringBoot 를 활용중입니다 

```
@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
		MySystemInfo info =  ctx.getBean("MySystemInfo" , MySystemInfo.class);
		
	}
		
}

```
잠깐 Boot 를 설명드리면 기본적으로 Srping 은 런타임 직후 실행되는 메인 메서드가 없습니다 boot 또한 형식적 표현이지 저기 안에서 무엇인가 진행할 수 없습니다 
그렇기에 우리는 ApplicationRunner 를 이용해서 런타임 직후 바로 호출되는 인터페이스와 run 메서드 재정의를 통해 Ioc 컨테이너를 생성해보겠습니다

```
@Override
public void run(ApplicationArguments args) throws Exception {

  ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
  MySystemInfo info =  ctx.getBean("MySystemInfo" , MySystemInfo.class);
  
}

```
첫줄 `ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");` 로 인해서 외부 메타데이터 bean.xml 을 읽을려고 하는것입니다 이를 통해서 Ioc 컨테이너 생성하고 Bean 을 Ioc 컨테이너가 관리하게 만들것입니다 

그리고 두번째 줄 `MySystemInfo info =  ctx.getBean("MySystemInfo" , MySystemInfo.class);` 을 통해서 ctx 의 getBean 이라는 메서드를 통해서 Bean 을 가져와서 MySystemInfo 탑의 객체를 주입합니다 그럼 이 코드의 결과는 MySystemInfo 생성자의 즉시 호출이 될것인데 결과를 보면 

## 호출결과 
```
MySystemInfo Bean 생성
```

역시 console 결과대로 생성자가 바로 호출되는것을 볼 수 있다 우리는 이제까지 java 에서 new 를 사용하지 않고는 생성자 호출을 할 수 없었지만 (정확히는 객체생성)
지금은 new 연산자 없이 `ctx.getBean("MySystemInfo" , MySystemInfo.class);` 코드 한줄로 객체를 만들고 생성자를 만들어서 안에 있는 내용을 호출하는것을 보여주었다 
이것이 제어의 역전이 일어난 것입니다





















