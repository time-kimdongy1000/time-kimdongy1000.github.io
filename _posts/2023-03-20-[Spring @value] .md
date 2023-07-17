---
title: Spring Ioc @Value
author: kimdongy1000
date: 2023-03-20 14:20
categories: [Back-end, Spring - Core]
tags: [ "@value" ]
math: true
mermaid: true
---

## @Value 의 사용법 
일반적으로 외부화된 속성을 주입할때 사욯합니다 


SpringRestartApplication.java
```

package com.cybb.main;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner {

	@Value("${catalog.name}")
	private String catalog_name;

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		System.out.println("catalog_name : " + catalog_name);

	}
}




```

일반적으로 이렇게 사용한다 그러면 어디선가 catalog.name key 로 된 값을 주입해주는데 보통은 properties 에서 이런 작업을 많이한다 예를 들어서 보자 

application.properties
```

catalog.name=Spring_boot_catalog_name


```
우리는 boot 프로젝트를 만들면 기본적으로 제공되는 properties 파일을 제공받는데 여기서 이렇게 값을 넣어주게 된다 만약 여기에 값이 없으면 default 값을 세팅할 수 있는데 

## value default 값

```
@Value("${catalog.name : non_catalog_name}")
private String catalog_name;

```
우리는 application.properties 내용을 지우고 여기서 : 을 기준으로 디폴트 값을 넣어주게 되면 값이 주입값이 없을때 default 값을 넣어주게 된다 
빈문자열 또한 넣을 수 있는데 이때는 @Value("${catalog.name :}") 이렇게 그냥 아무값도 않넣고 콜론만 넣어주면된다 



## 만약 default 값도 없다면 
```
Could not resolve placeholder 'catalog.name ' in value "${catalog.name }"

```
이렇게 catalog.name 에 대한 주입값을 찾을 수 없기 때문에 프로그램이 종료 된다 이때는 그냥 : 라도 넣어주는것에 좋다 


## 다른 properties 에서 값을 세팅할때 
spring 은 다양한 properties 파일을 제공하고 읽어낼 수 있다 지금은 기본 properties 파일에 적었지만 보통은 properties 를 분리하는 경향이 많다 큰 프로젝트일 수록 말이다 
그럼 이번엔 다른곳에 properties 파일을 세팅하고 읽어내는 방식을 생각하자 


app.properties
```

APP_NAME=Spring_Boot_App

```

이번엔 app.properties 에 값을 세팅하고 이 값을 읽어보자 

SpringRestartApplication.java
```
package com.cybb.main;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner {

	@Value("${catalog.name:}")
	private String catalog_name;

	@Value("${APP_NAME:}")
	private String app_name;

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		System.out.println("catalog_name : " + catalog_name);
		System.out.println("app_name : " + app_name);
		

	}
}

```
우리는 app_name @value 로 선언했지만 이번에는 app.properties 에 값을 세팅을 해보자 

```
catalog_name : Spring_boot_catalog_name
app_name : 

```

결과는 나오지 않았다 기본적으로 설정을 해주지 않으면 프러퍼티는 application.properties 만 읽게 된다 그렇기에 다른 properties 를 읽게끔 설정을 해주어야 하는데 

`@PropertySource("classpath:/app.properties")` 클래스 상단에 선언해주자 

```

@SpringBootApplication
@PropertySource("classpath:/app.properties")
public class SpringRestartApplication implements ApplicationRunner {}

```





















