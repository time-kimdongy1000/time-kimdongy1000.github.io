---
title: Spring Ioc 와 java Bean
author: kimdongy1000
date: 2023-03-12 11:22
categories: [Back-end, Spring - Core]
tags: [Bean]
math: true
mermaid: true
---

## Java 기반 컨테이너 구성
우리는 앞에서 계속 xml 방식으로 bean 을 생성하고 컨테이너를 생성후 bean 을 직접 주입했다 이런 방식은 전통적인 xml 주입방식으로 레거시 프로젝트 이외에는 거진 bean 관리를 이렇게 하지 않고 
java code 방식으로 bean 을 구성한다 이번시간부터는 이런 방식에 대해서 공부를 진행을 할려고 합니다


## @Bean 
처음에 공부할것은 @Bean 에 대해서 알아보겠습ㅂ니다 
이 애노테이션은 spring Ioc 컨테이너 에서 관리할 새 개체를 인스턴스화 , 구성 및 , 초기화함을 나타내는데 사용합니다 우리가 앞에서 사용한 xml `<bean>` 과 동일한 역활을 하는것입니다 


## @Configuration 
이 애노테이션은 메서드 단위에서 정의되는 @Bean 하고는 달리 @Configuration 는 클래스 단위에서 정의해서 이 class 가 빈 정의 소스 코드라는 것을 나타냅니다 그래서 우리는 가장 간단한 
@Bean 과 @Configuration 을 나타내면 



MySystemInfo.java
```

package com.cybb.main;

public class MySystemInfo {
	
	
	public MySystemInfo() {
		System.out.println("나의 시스템은 정상입니다");
	}

}


```

AppConfig.java
```

package com.cybb.main;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {
	
	@Bean
	public MySystemInfo info () {
		
		return new MySystemInfo();
	}

}


```


SpringRestartApplication.java
```

package com.cybb.main;


import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
		MySystemInfo info =  ctx.getBean(MySystemInfo.class);

	}
	
	

}


```

이렇게 쓰일것인데 이제 우리는 AppConfig 에 MySystemInfo 타입을 정의해둔뒤 이를 호출해서 불러오는 것이 된다 그럼 실제로 Runner 에서 한번더 부를때에는 앞에서는 
ClassPathXmlApplicationContext 를 읽어서 xml 을 읽은 다음 컨테이너를 만들었다면 이제는 xml 이 아닌 애노테이션 기반의 java 코드를 읽어야 함으로 
AnnotationConfigApplicationContext 사용하게 될것이다 그런데이를 실행하면 결과가 조금 의야하게 나올 수 있을것이다 우리는 분병 생성자를 한번만 호출할것인데


```

나의 시스템은 정상입니다
[2m2023-03-13 09:51:17.808[0;39m [32m INFO[0;39m [35m24664[0;39m [2m---[0;39m [2m[           main][0;39m [36mo.s.b.w.embedded.tomcat.TomcatWebServer [0;39m [2m:[0;39m Tomcat started on port(s): 8080 (http) with context path ''
[2m2023-03-13 09:51:17.816[0;39m [32m INFO[0;39m [35m24664[0;39m [2m---[0;39m [2m[           main][0;39m [36mcom.cybb.main.SpringRestartApplication  [0;39m [2m:[0;39m Started SpringRestartApplication in 1.501 seconds (JVM running for 2.18)
나의 시스템은 정상입니다


```

이렇게 두번 호출되는것을 볼 수 있는데 이는 Spring 특유의 Bean 스캔 시스템이 달려 있다 Bean 스캔은 런타임시 스프링은 정의되어 있는 모든 Bean 을 감지해서 이미 IoC 에 넣어두고 기동을 시키고 사전에 검사까지 진행을 하게 된다 

```

@Bean
public MySystemInfo info () {
    
    return new MySystemInfo();
}

```
이때 bean 스캔도중 이 코드를 만나게 되면 스프링은 컨터이너에 이미 MySystemInfo 객체를 만들어 놓는다 이게 1차 생성자 호출 

2번째 생성자 호출은 Runner 에서 한번더 읽게끔 소스를 만들었다 `ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);`
이 소스로 인해서 한번더 bean 을 읽어서 객체를 생성한다 그때 생성자가 한번더 호출되면서 콘솔에 두번씩 찍히게 된다 이 bean 스캔에 대해서는 따로 자세하게 다룰 때가 있을것이다 
