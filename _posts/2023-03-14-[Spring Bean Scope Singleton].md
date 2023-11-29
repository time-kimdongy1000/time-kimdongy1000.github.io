---
title: Spring Ioc 와 Bean Scope Singleton
author: kimdongy1000
date: 2023-03-14 10:14
categories: [Back-end, Spring - Core]
tags: [ Bean Scope Singleton]
math: true
mermaid: true
---

## Bean 의 주소값
Spring 의 Bean 의 범위는 총6가지가 존재한다 이 6개는 각각의 범위로 생성할 수 있으며 아무런 설정을 하지 않은채로 Bean 을 만들면 기본적으로는 싱글톤 패턴으로 만들게 된다 

## singleton 
특별한 Bean 설정을 하지 않으면 가지는 기본 Bean 의 범위로 Spring Ioc 컨테이너에 대한 단일 객체 인스턴스에 단일 빈 범위를 지정합니다 
즉 싱글톤 범위로 지정된 Bean 은 IoC 컨테이너에 정확히 단 한개의 인스턴스를 생성하게 됩니다 그럼 소스코드를 보자 

```
public class MySystemInfo {
	
	public String returnMsg() {
	
		return "시스템은 정상입니다";
	}

}
```


```
@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{
	
	@Autowired
	private System1 system1;
	
	@Autowired
	private System2 system2;
	
	@Autowired
	private System3 system3;
	
	
	
	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		
		System.out.println(system1.returnMsg());
		System.out.println(system2.returnMsg());
		System.out.println(system3.returnMsg());
	}
}
```


```
@Component
public class System1 {
	
	
	@Autowired
	private MySystemInfo info;
	
	public String returnMsg() {
		
		System.out.println(info);
		
		return "1번시스템은" + info.returnMsg();
	}

}
```

```
@Component
public class System2 {
	
	@Autowired
	private MySystemInfo info;
	
	public String returnMsg() {
		
		System.out.println(info);
		
		return "2번시스템은" + info.returnMsg();
	}
}
```

```
@Component
public class System3 {
	
	@Autowired
	private MySystemInfo info;
	
	public String returnMsg() {
		
		System.out.println(info);
		
		return "3번시스템은" + info.returnMsg();
	}
}
```

```
@Configuration
public class AppConfig {
	
	@Bean
	public MySystemInfo info() {
	
		return new MySystemInfo();
	}
}
```


소스가 많지만 정리해보면 MySystemInfo 를 먼저 정의하고 AppConfig 에 Bean 으로 정의를 한다 그리고 각 System1 , System2 , System3 를 정의해서 
private MySystemInfo info; 를 @Autowired 로 상속을 받고 메서드 하나를 만드는데 이때 이 info 의 주소값을 각각 찍어보라고 진행을 한것이다 
그래서 main 을 보면


```
com.cybb.main.MySystemInfo@3468ee6e
1번시스템은시스템은 정상입니다
com.cybb.main.MySystemInfo@3468ee6e
2번시스템은시스템은 정상입니다
com.cybb.main.MySystemInfo@3468ee6e
3번시스템은시스템은 정상입니다

```

객체를 찍으면 주소값이 나옴으로 이렇게 다 동일한 주소값이 나오는것을 확인할 수 있다 이렇게 이렇게 Bean 을 하나 만들고 여러곳에서 객체를 주입하더라도 
객체의 주소값은 동일하기에 동일한 단 하나의 인스턴스를 서로다른 곳에서 사용하고 있다 그래서 다음과 같은 상황도 볼 수 있는데 예를 들어서

## 싱글톤 객체의 메모리 위치 공유
```

package com.cybb.main;

public class MySystemInfo {
	
	private int mySystemInfo;
	
	public MySystemInfo() {
		mySystemInfo = 0;
	}
	
	
	public String returnMsg() {
	
		return "시스템은 정상입니다";
	}
	
	public void plusMySystemInfo() {
		mySystemInfo++;
	}
	
	public int getMySystemInfo() {
		return mySystemInfo;
	}	
}

```

필드하나를 만들고 메서드를 호출할때마다 필드값을 ++ 하는 메서드를 만들었습니다 

```
public String returnMsg() {
	System.out.println(info);
	info.plusMySystemInfo();
	
	return "1번시스템은" + info.returnMsg();
}

```

그리고 각 시스템마다 returnMsg 호출할때마다 info.plusMySystemInfo(); 를 호출하는 메서드를 만들어 주었습니다 


```
@Autowired
private MySystemInfo info;

public static void main(String[] args) {
	SpringApplication.run(SpringRestartApplication.class, args);
}

@Override
public void run(ApplicationArguments args) throws Exception {
	
	System.out.println("최초 값 : " + info.getMySystemInfo());
	
	
	System.out.println(system1.returnMsg());
	System.out.println("첫번째 호출 : " + info.getMySystemInfo());
	
	
	System.out.println(system2.returnMsg());
	System.out.println("두번째 호출 : " + info.getMySystemInfo());
	
	System.out.println(system3.returnMsg());
	System.out.println("세번째 호출 : " + info.getMySystemInfo());
}
	
```

메인에서 한번 호출할때마다 값을 찍어보면 이는 메모리 주소가 한개인 인스턴스를 서로 공유하기에 

```
최초 값 : 0
com.cybb.main.MySystemInfo@4976085
1번시스템은시스템은 정상입니다
첫번째 호출 : 1
com.cybb.main.MySystemInfo@4976085
2번시스템은시스템은 정상입니다
두번째 호출 : 2
com.cybb.main.MySystemInfo@4976085
3번시스템은시스템은 정상입니다
세번째 호출 : 3

```

이렇게 찍히게 됩니다 이것이 스프링이 채택하는 가장 기본인 싱글톤 Bean 패턴입니다