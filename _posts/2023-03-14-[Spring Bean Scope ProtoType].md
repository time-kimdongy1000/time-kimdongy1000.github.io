---
title: Spring Ioc 와 Bean Scope ProtoType
author: kimdongy1000
date: 2023-03-14 13:39
categories: [Back-end, Spring - Core]
tags: [ Bean Scope ProtoType]
math: true
mermaid: true
---

## ProtoType 
앞 글에서 Srping이 Bean 생성할때 기본 디폴트 생성 정책은 싱글톤 패턴입니다 이 싱글톤 패턴은 여러개의 Bean 을 주입한다고 할지라도 
하나의 인스턴스에 단 한개의 동일한 메모리 주소를 가지고 있는 것입니다 

그렇기에 안에 있는 필드 변경값을 계속 공유하고 있는 모습을 코드로 보았습니다 이번 시간에는 매번 새로운 객체를 생성하는 정책인 
프로토타입에 대해서 공부를 해보겠습니다 

소스는 어제거 그대로 가져오시되 Bean 을 생성할때 정책을 설정해주시면됩니다 


특별한 Bean 설정을 하지 않으면 가지는 기본 Bean 의 범위로 Spring Ioc 컨테이너에 대한 단일 객체 인스턴스에 단일 빈 범위를 지정합니다 
즉 싱글톤 범위로 지정된 Bean 은 IoC 컨테이너에 정확히 단 한개의 인스턴스를 생성하게 됩니다 그럼 소스코드를 보자 



```
@Configuration
public class AppConfig {
	
	@Bean
	@Scope(scopeName = "prototype")
	public MySystemInfo info() {
	
		return new MySystemInfo();
	}

}


```
@Scope(scopeName = "prototype") 이렇게 Bean 위에 이렇게 붙여주면 이제 이 Bean 은 주입할때마다 완전 새로운 객체를 생성하게 됩니다 

```

최초 값 : 0
com.cybb.main.MySystemInfo@3d90eeb3
1번시스템은시스템은 정상입니다
첫번째 호출 : 0
com.cybb.main.MySystemInfo@1db87583
2번시스템은시스템은 정상입니다
두번째 호출 : 0
com.cybb.main.MySystemInfo@7fb53256
3번시스템은시스템은 정상입니다
세번째 호출 : 0


```
그래서 현재 info 의 메모리 주소는 전혀 다르게 나와 있고 우리가 앞에서 호출하는 plusMySystemInfo 메서드는 후행연산자 이기 때문에 계속해서 0만 찍혀 있는것을 알 수 있습니다 

## 프로토 타입의 특징
스프링은 싱글톤 패턴의 Bean 의 생명주기 전체를 관리하지만 프로토 타입은 특정 생명주기만 관리하게 됩니다 
그래서 Bean 의 생성 , 소멸의 콜백함수를 설정해 놓는다고 할지라도 프로토타입의 Bean 은 경우에 따라서 호출되지 않을 수 있습니다 
그래서 이런 Bean 들은 소멸되지 않고 리스트업 해서 직접 해제를 해야 합니다 
그리고 이 프로토타입은 스레드 세이프 하지 않습니다 즉 여러 사람이 한번에 들어오는 시스템은 늘 thread safe 이슈가 발생할 수 있습니다 