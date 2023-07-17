---
title: Spring Ioc 와 Component Scan
author: kimdongy1000
date: 2023-03-12 11:22
categories: [Back-end, Spring - Core]
tags: [Component Scan]
math: true
mermaid: true
---

## Component Scan
우리는 앞의 예제 마지막에소 왜 생성자가 두번 호출되는지에 대해서 간략하게 말하면서 Bean Scan 에 대해서 말한적이 있다 Bean Scan Componet Scan 동일한 뜻으로 사용되며 
실제로 spring 은 실행이 될때 런타임 시점이세 감지되는 모든 Bean 을 이미 IoC 컨테이너에 미리 넣어두게 된다 우리가 직접 설정할 수도 있지만 Spring 이 권하는 방식은 이미 
선언되어 있는 Bean Scan 룰을 따르길 바란다 그럼 Spring 에서 이미 만들어 놓은 룰을 한번 보자 

## @SpringBootApplication
Boot 기준으로 설명을 드리면 우리가 앞에서 제일 먼저 볼 수 있는 애노테이션이다 

```

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}

```

이 애노테이션을 까보면 이와 같은 여러개의 또다른 애노테이션이 있는데 우리가 살펴볼것은은 @ComponentScan 을 살펴보면된다 
여러개가 붙어있긴 하지만 excludeFilters 특정 Bean 을 ComponentScan 에서 제외하겠다는 뜻이다 즉 저기에 들어가는 특정 Bean 들은 스캔등록으로 사용하지 않겠다는 뜻이다 
특징적으로 Filter 를 Bean 으로 등록을 하지 않겠다는 뜻인데 기본적으로 사용자 정의 Filter 같은 경우중에서 TypeExcludeFilter 와 AutoConfigurationExcludeFilter 로 구현이 되어 있는 
커스텀 필터는 기본적으로 Bean 으로 등록을 하지 않겠다는 뜻이다 

그외에는 전부 Bean 으로 등록을 해주는것인데 

```

@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
String[] scanBasePackages() default {};

```
scanBasePackages 라고 basePackages 가 있다 이 basePackages 는 우리가 처음 프로젝트를 만들때 기본 패키지를 등록하게 되어 있다 


![spring-base-package1](https://user-images.githubusercontent.com/58513678/224592594-7d0d00cf-a4bd-474c-9ef2-1cb43d25349d.jpg)

아래에 보면 Package 가 존재하는데 이게 기본 패키지가 되는것이다 즉 spring 에서 감지하는 basePackages 에 포함이 되는것이다 


그럼 만약에 BasePackage 를 벗어나는 Bean 을 선언했을때에는 어떤 현상이 일어나는지 살펴보자 

## 사용자 정의 Bean 스캔 

MySystemInfo2.java
```

package com.cybb.main2;

public class MySystemInfo2 {
	
	public MySystemInfo2() {
		
		System.out.println("MySystemInfo2");
	}

}


```

AppConfig2.java
```

package com.cybb.main2;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig2 {
	
	@Bean
	public MySystemInfo2 info2() {
		
		return new MySystemInfo2();
	}

}


```
BasePackages 는 com.cybb.main 이지만 우리는  com.cybb.main2 코드를 넣고 기동을 해보자 만약 Bean 스캔이 되면 생성자가 호출이 될것이다 
역시나 생성자 호출이 일어나지 않는다 그럼 강제적으로 읽혔을때 한번 살펴보자


```

package com.cybb.main;


import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import com.cybb.main2.AppConfig2;
import com.cybb.main2.MySystemInfo2;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class , AppConfig2.class);
		MySystemInfo info =  ctx.getBean(MySystemInfo.class);
		MySystemInfo2 info2 =  ctx.getBean(MySystemInfo2.class);

	}
	
	

}


```
강제로 읽게 하는 방법은 AnnotationConfigApplicationContext 에 우리가 정의한 Configuration 파일을 읽게 시키면된다 

```
SpringRestartApplication  [0;39m [2m:[0;39m Started SpringRestartApplication in 2.523 seconds (JVM running for 3.717)
나의 시스템은 정상입니다
MySystemInfo2

```

이때의 결과는 다르게 나타난데 Bean 을 읽은것이다 즉 런타임시에는 basePackage 에 포함되지 않았기에 스캔이 되지 않았지만 
강제로 읽게 시켜서 Ioc 컴포넌트에 넣었을때는 Bean 이 생겨나서 생성자가 호출되는것을 확인했다 그럼 이제 우리는 이 패키지를 자동으로 스캔되게끔 설정을 해보자 

소스는 간단하다 
`@SpringBootApplication(scanBasePackages = "com.cybb.*")` 이렇게 좀더 상위 레벨로 Bean 스캔 범위 설정을 해두면 런타임시 Ioc 컨테이너에 Bean 을 생성해서 넣어둔다 
그것을 알 수 있는게 생성자가 각 2번씩 호출되는것을 알 수 있다 

```
나의 시스템은 정상입니다
MySystemInfo2


```

그러면 우리는 우리가 원하는 대로 component scan 과 사용자 설정 Bean 스캔 범위 설정에 대해서 공부해보았다 












