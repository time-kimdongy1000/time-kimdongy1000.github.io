---
title: Spring Ioc 와 Bean @Primary
author: kimdongy1000
date: 2023-03-12 15:21
categories: [Back-end, Spring - Core]
tags: [ Primary ]
math: true
mermaid: true
---

## Bean 의 기본설정 
기본적으로 동일한 이름의 Bean 은 단 한개만 만들 수 있습니다 예를 들어서 다음과 같은 소스코드가 있을때는 에러가 발생합니다 

```
package com.cybb.main;

public class MySystemInfo {
	
	public MySystemInfo() {
		
		System.out.println("MySystemInfo 객체 생성");
	}
}

```

```

package com.cybb.main;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {
	
	@Bean
	public MySystemInfo info1() {
		
		return new MySystemInfo();
	}
	
	@Bean
	public MySystemInfo info2() {
		
		return new MySystemInfo();
	}
}
```

```

package com.cybb.main;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{
	
	@Autowired
	private MySystemInfo info;

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		
	}
}
```

지금 AppConfig.java 에서 동일한 객체 타입을 서로 다른 메서드로 호출해서 Bean 으로 만들려고 합니다 이때 기동을 하게 되면 다음과 같은 에러를 발견하게 됩니다 

```

***************************
APPLICATION FAILED TO START
***************************

Description:

Field info in com.cybb.main.SpringRestartApplication required a single bean, but 2 were found:
	- info1: defined by method 'info1' in class path resource [com/cybb/main/AppConfig.class]
	- info2: defined by method 'info2' in class path resource [com/cybb/main/AppConfig.class]


Action:

Consider marking one of the beans as @Primary, updating the consumer to accept multiple beans, or using @Qualifier to identify the bean that should be consumed

```

실패 원인은 SpringRestartApplication 에서는 하나의 bean 만 있으면 되는데 Ioc 컨테이너에 현재 2개의 동일한 Bean 이 존재하는 뜻입니다 
그래서 해결방식으로 bean 을 만들때 @Primary 을 붙여주거나 아니면  @Qualifier 를 활용하라는 뜻입니다 

## @Primary
Pri 의 접두사는 유일하다는 뜻입니다 이 영어 단어의 어근을 살펴보면 prim 이 있는데 이는 첫 번째의 라는 뜻으로 변화형으로 pri 가 되었다는데 이 어근의 뜻 대로 유일하다는 뜻입니다 @Primary 는 Bean 을 정의할때 붙여주는것으로 

```

@Bean
@Primary
public MySystemInfo info1() {
	
	
	return new MySystemInfo();
}

@Bean
public MySystemInfo info2() {
	
	return new MySystemInfo();
}

```

@Bean 을 정의할때 아래에 적어주기만 하면된다 그리고 기동을 하게 되면 아까 오류는 사라지게 된다 하지만 실제로 이게 올바르게 작동되고 있는지 조금의문이다 그래서 이와같이 코드를 작성해보자


```

@Bean
@Primary
public MySystemInfo info1() {
	
	return new MySystemInfo("1번 시스템 호출");
}

@Bean
public MySystemInfo info2() {
	
	return new MySystemInfo("2번 시스템 호출");
}

```

AppConfig 는 이렇게 수정하고 

```

package com.cybb.main;

public class MySystemInfo {
	
	private final String msg;
	
	public MySystemInfo(String msg) {
		this.msg = msg;
	}
	
	public String getMsg() {
		return msg;
	}

}


```
MySystemInfo 는 들어온 값 한번의 msg 를 생성자를 통해 세팅을해서 그 값을 get 할것이다 만약 @Primary 1번 시스템 호출 이 들어올것이다 


```

package com.cybb.main;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{
	
	@Autowired
	private MySystemInfo info;
	
	
	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		
		System.out.println(info.getMsg());

	}
}
```

결과는 1번 시스템 호출 단 한번만 호출되었다 Bean 을 생성할때는 2개를 생성하지만 결국 사욯할려고 할때는 기본적으로 @Primary 이 있는 Bean 타입을 우선하고 
해당 Bean 을 주입해주는 결과를 볼 수 있는것이다 

@Primary 는 @Bean 타입에만 한정되지 않습니다 그래서 이와 같이도 사용할 수도 있습니다 

## @Component @Primary 의 조합 
```

package com.cybb.main;

public interface MySystemInfo {
	
	public String mySystemInfo (String msg);
}
```

```

package com.cybb.main;

import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;

@Component
@Primary
public class Info1 implements MySystemInfo{

	@Override
	public String mySystemInfo(String msg) {
		return msg + "Info1 번이 호출되었습니다.";
	}
}
```

```

package com.cybb.main;

import org.springframework.stereotype.Component;

@Component
public class Info2 implements MySystemInfo {

	@Override
	public String mySystemInfo(String msg) {
		return msg + "2번이 호출되었습니다";
	}
}

```

이렇게 MySystemInfo 의 인터페이스를 만들고 이를 구현하는 구현체 2개를 만들어서 @Component 화 해서 Bean 으로 만듭니다 이때 Info1 클래스에는 @Primary 를 선언했습니다 


SpringRestartApplication.java
```

package com.cybb.main;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{
	
	@Autowired
	private MySystemInfo info;
	
	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		System.out.println(info.mySystemInfo("@Component 를 테스트 중입니다"));
	}
}
```

@Autowired 를 보면 MySystemInfo 타입을 지정했다 이때 인터페이스는 실제 Bean 으로 생성되지 않기에 구현체 중에서 이 타입이 있는지 살펴보게 된다 그러면 
2개의 Bean 이 보일텐데 마찬가지로 @Primary 클래스 레벨에 선언되어 있는 Info1 클래스가 Bean 으로 주입되어 사용될것입니다 
결과를 살펴보면

```
@Component 를 테스트 중입니다Info1 번이 호출되었습니다.

```

이런 결과를 볼 수 있습니다

우리는 동일레벨 중에서 어떤 Bean 을 우선해서 주입할 수 있는지에 대해서 배웠습니다 이게 Bean 생성 레벨에서 정의되는것으로 다음에는 Bean 주입할때 사용하는 
@Qualifier 에 대해서 알아보겠습니다.