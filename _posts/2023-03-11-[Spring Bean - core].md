---
title: Spring Ioc 와 Bean1
author: kimdongy1000
date: 2023-03-12 08:23
categories: [Back-end, Spring - Core]
tags: [Bean]
math: true
mermaid: true
---

## 빈 개요
Ioc 컨테이너는 하나 이상의 Bean 을 관리합니다 이러한 Bean 은 우리가 앞서서본 xml 메타데이터를 읽어서 생성하는것이 전통적인 방식입니다
Bean 의 명명규칙은 표준 java 규약을 사용하는것입니다 소문자로 시작하고 카멜케이스로 작성되는것이 규약입니다 

## 생성자를 통한 bean 주입 
이번시간에는 생성자를 통해서 bean 을 주입하는 방식을 알아볼것인데 소스가 조금 길수 있다 

## bean.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd">

    <bean id = "MySystemInfo" class = "com.example.demo.MySystemInfo">
        <constructor-arg ref = "MyCpuInfo" />

    </bean>

    <bean id ="MyCpuInfo" class="com.example.demo.MyCpuInfo">
        <constructor-arg name="cpuName" value="Intel" />
        <constructor-arg name="use" value="100" />
    </bean>

</beans>
```

## MySystemInfo
```
public class MySystemInfo {

    private final MyCpuInfo myCpuInfo;

    public  MySystemInfo (MyCpuInfo myCpuInfo){
        this.myCpuInfo = myCpuInfo;
    }

    public void MySystemInfoSay(){
        System.out.println("cpuName : " + myCpuInfo.getCpuName());
        System.out.println("use : " + myCpuInfo.getUse());
    }
}

```

## MyCpuInfo
```
public class MyCpuInfo {

    private String cpuName;
    private int use;

    public MyCpuInfo(String cpuName , int use){
        this.cpuName = cpuName;
        this.use = use;

    }

    public String getCpuName() {
        return cpuName;
    }

    public void setCpuName(String cpuName) {
        this.cpuName = cpuName;
    }

    public int getUse() {
        return use;
    }

    public void setUse(int use) {
        this.use = use;
    }
}

```

우리는 bean 을 두가지 만들것이다 MySystemInfo Bean 하고 MyCpuInfo 두가지 bean 을 만들것이다 이떄 지금 처음 bean 을 만들었을대보다 다소 길어진 측면이 있지만 하나씩 실펴보자 먼저 MyCpuInfo를 소스를 먼저 보자 

두개의 필드 cpuName , use 가 존재하고 이를 주입받는곳은 생성자에서 주입을 받고 마지막으로 getter , setter 이 존재한다 이때 이 cpuName , use 를 주입해주는곳은 
바로 bean.xml 에 ` <constructor-arg name="cpuName" value="Intel" /> <constructor-arg name="use" value="100" />` 이 두가지가 되는것이다 
constructor-arg 생성자 파라미터라는 뜻으로 생성자 안에 들어오는 파라미터의 값을 세팅해줄때 사용하는 구성 메타데이터 태그이다 즉 우리는 MyCpuInfo Bean 이 생성될때 
각 필드 cpuName 에는 Intel 이라는 use 에는 100이라는 값을 세팅을 해준다는 뜻이 된다 

그럼 MySystemInfo bean 정의는 무엇일까 역시나 bean 을 만드는데 `<constructor-arg ref = "MyCpuInfo" />` ref 라는 값이 들어가 있는데 이 ref 뜻은 bean 이 생성이 되어 있으면 그것을 참조하겠다는 뜻이다 즉 MySystemInfo 는 MyCpuInfo 를 자신의 페이지에서 사용하기 위해서 이 MyCpuInfo을 생성자로 주입받는 상황인것이다 그럼 이제 Ioc 컨테이너를 만들어보자

## Ioc 컨테이너 생성

```
@SpringBootApplication
public class SpringCoreApplication implements ApplicationRunner {

	public static void main(String[] args) {
		SpringApplication.run(SpringCoreApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
		MySystemInfo mySystemInfo = ctx.getBean("MySystemInfo" , MySystemInfo.class);
		mySystemInfo.MySystemInfoSay();
	}
}


```
컨테이너는 지난시간 만들어보았으니 쉬울것이다 classpath 상의 xml 을 읽고 MySystemInfo 의 메서드를 호출만 해주면되는것이다 그럼 결과는 다음과 같이 나올것이다

```

cpuName : Intel
use : 100

```

우리는 이렇게 생성자에 있는 bean 타입에 대해서 주입하는 방법을 공부해보았다 