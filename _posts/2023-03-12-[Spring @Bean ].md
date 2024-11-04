---
title: Spring Ioc @Bean
author: kimdongy1000
date: 2023-03-12 11:22
categories: [Back-end, Spring - Core]
tags: [IoC , '@Bean']
math: true
mermaid: true
---

우리는 앞에서 xml 을 통한 bean 정의서를 만들고 그것을 IoC 컨테이너에 주입하는 방식으로 진행을 했다 이때 xml bean 정의서의 가장 큰 단점 중 한 가지는 바로 오타 또는 기타 오류 검증이 안된다는 단점이 있다 다시 앞의 bean.xml 파일을 보자

## bean.xml
```

<?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
	  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	  xmlns:context="http://www.springframework.org/schema/context"
	  xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd">
		
	  	<bean id = "student" class="com.cybb.main.Student">
	  		<property name="name" value="time"/>
	  		<property name="age" value="20"/>
	  	</bean>
	  	
	  	<bean id = "classRoomA" class="com.cybb.main.ClassRoomA">
	  		<constructor-arg ref = "student"/>
	  	</bean>
    </beans>

```
이때 이 이 태그에서 오류를 한 가지라도 발생하면 xml 파일을 읽지 못하거나 bean 을 생성하지 못합니다 단적인 예로 `<bean id = "student" class="com.cybb.main.Student">`에서
class를 오타를 내어서 `<bean id = "student" class="cOm.cybb.main.Student">` 이렇게 bean 생성될 위치를 올바르게 명시하지 못하면 오류가 발생하게 됩니다

그래서 나온 게 java 어노테이션 기반의 bean 정의가 등장을 하게 됩니다

## 어노테이션 기반 
어노테이션은 기본적으로 @로 시작하는 java 명령어입니다 이 어노테이션은 소스 안에 이 어노테이션을 붙였을 시 어떤 행동을 하라는 정의가 이미 소스 안에 다 만들어져 있기 때문에
그냥 바로 가져다 쓰면 됩니다

## 애노테이션 기반 bean 생성 

```
@Configuration
public class BeanConfig {
	
	@Bean
	public Student student() {
		
		Student student = new Student();
		student.setName("time");
		student.setAge(20);
		
		return student;	
	}
	
	@Bean
	public ClassRoomA classRoomA() {
		
		ClassRoomA classRoomA = new ClassRoomA(student());
		
		return classRoomA;
	}

}


```

이렇게 java 기반으로 bean 객체 정의서를 만들 수 있습니다 이때 `@Configuration` 이 클래스 파일은 bean 설정 파일이라는 것을 명시해 줍니다 그리고 각 bean 이 될 애들에 대해서는
`@bean` 어노테이션을 붙여서 이를 Ioc 가 관리하는 bean 객체가 되는 것입니다 자 그럼 이들을 불러오는 방법은 아래와 같다

## 어노테이션 기반 bean 을 Ioc 컨테이너에 담는 방법
```

ApplicationContext applicationContext = new AnnotationConfigApplicationContext(BeanConfig.class);
ClassRoomA roomA = applicationContext.getBean(ClassRoomA.class);
roomA.classRoomAInStudent();

```
우리는 앞에서 ClassPathXmlApplicationContext 로 bean.xml 파일을 읽었다면 이제는 다른 구현체 AnnotationConfigApplicationContext를 사용해서 Ioc 컨테이너에 bean 을 담게 됩니다
이 이후는 앞의 내용과 동일함으로 생략을 할 텐데 여기서 궁금증이 생길 것입니다


## @Bean 태그가 있는것과 없는것의 차이점 

```

@Bean
public Student student() {
	
	Student student = new Student();
	student.setName("time");
	student.setAge(20);
	
	return student;
	
}


public Student student() {
	
	Student student = new Student();
	student.setName("time");
	student.setAge(20);
	
	return student;
	
}

```
이 두 개의 소스는 어찌 보면 위에 @Bean 어노테이션 차이 말고는 하는 역할은 똑같습니다 하지만 이 객체가 어디로 들어가냐에 따라서 그 이후 상황은 완전히 달라지게 됩니다
@Bean 태그가 붙은 객체는 Ioc 컨테이너에서 관리가 되고 그게 없는 객체는 그냥 GC(Garbage Collection)에서 관리가 될 것입니다
그리고 당장 눈에 볼 수 있는 차이점은 @Bean 이 붙은 객체는 여러 번 호출해도 단 하나의 객체가 계속해서 반환이 되고 (Bean 생성 전략) 그렇지 않은 객체는 계속해서 새로운 객체를 생성할 것입니다 그를 비교할 수 있는 코드는 아래와 같습니다

```

ApplicationContext applicationContext = new AnnotationConfigApplicationContext(BeanConfig.class);
ClassRoomA roomA = applicationContext.getBean(ClassRoomA.class);

Student student1 = applicationContext.getBean(Student.class);
Student student2 = applicationContext.getBean(Student.class);
Student student3 = applicationContext.getBean(Student.class);

BeanConfig beanConfig = new BeanConfig();

Student student4 = beanConfig.student();
Student student5 = beanConfig.student();
Student student6 = beanConfig.student();

System.out.println(student1.hashCode());
System.out.println(student2.hashCode());
System.out.println(student3.hashCode());

System.out.println(student4.hashCode());
System.out.println(student5.hashCode());
System.out.println(student6.hashCode());

```
hashCode는 객체를 구별할 때 유용하게 쓰일 수 있으며 같은 값을 반환하는 객체는 동일한 객체임을 알려줍니다 즉 Ioc 컨테이너에서 꺼내는 모든 객체는
동일한 객체를 나타냅니다 단 student 직접 호출한 4 5 6 객체는 전부 새로운 객체를 생성하기에 이들을 서로 다른 객체입니다