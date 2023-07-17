---
title: Spring Ioc 와 Bean Annotation
author: kimdongy1000
date: 2023-03-12 13:46
categories: [Back-end, Spring - Core]
tags: [Bean Annotation]
math: true
mermaid: true
---

## Bean Annotation
우리는 앞에서 xml 로 사용하는 방식을 애노테이션 방식으로 Bean 을 선언하는 방식을 바꾸어 보았고 
Bean 스캔 범위에 따라서 어떤 특정빈을 런타임시에 자동으로 Bean 이 스캔되어서 IoC 컨테이너에 들어가는것도 확인했습니다 
또한 런타임 스캔에 포함이 되어 있지 않더라도 난중에 추가할 수 있는 방식에 대해서 공부해 보았습니다 

그때마다 항상 빠지지 않는 코드가 있는데 바로 ApplicationContext 입니다 하지만 현업에는 Bean 주입을 이런식으로 진행되지 않습니다 
우리는 어제 시간으로 인해서 Component Scan 으로 이미 Bean 은 Ioc 컨테이너에 담겨 있는것을 확인했기에 
우리는 필요한곳에 주입만 해주면 되었기에 사실상 지난시간 같은 코드는 현업에서는 사용하지 않습니다 

## 현업에서는 이런식으로 Bean 정의 파일을 입력하지 않습니다 

```
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class , AppConfig2.class);
MySystemInfo info =  ctx.getBean(MySystemInfo.class);
MySystemInfo2 info2 =  ctx.getBean(MySystemInfo2.class);

```

그렇기에 다른 애노테이션을 활용해서 이제 Ioc 컨테이너도 우리 눈에서 없애버리도록 하겠습니다 물론 우리눈에만 안보이도록 할뿐입니다 

## @Component 
우리는 앞에서 Bean 을 새롭게 만들때 @Bean 이라는 애노테이션을 활용했지만 이는 메서드 선언 안에서만 활용되는 반면 @Component 클래스 레벨에서 Bean 을 만들떄 사용하는 애노테이션입니다 

```

package com.cybb.main;

import org.springframework.stereotype.Component;

@Component
public class MySystemInfo {
	
	
	public MySystemInfo() {
		System.out.println("시스템은 정상입니다");
	}

}


```

이런식으로 class 레벨에서 사용하며 이렇게 되면 MySystemInfo는 Bean 스캔시 자동으로 Ioc 컨테이너에 들어가게 됩니다 그럼 이를 다른 곳에서 주입받을려면 어떻게 사용해야 하느냐?

## @Autowired 

```

package com.cybb.main;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MySystem {
	
	private final MySystemInfo info;
	
	@Autowired
	public MySystem(MySystemInfo info) {
		this.info = info;
	}
	
	public void MysystemReturnMsg() {
		System.out.println("시스템2번" + info.returnMsg());
	}
}


```

이런식으로 @Autowired 활용해서 필드의 MySystemInfo 를 주입해주는것입니다 이때 @Autowired 주석의 일부분이지만 실제로 Ioc 컨테이너에 다녀와서 찾은 객체 타입 또는 이름을 가져와서 
주입해주는 식입니다 

자 그럼 실제로 Ioc 컨테이너 소스가 없어졌는지도 한번 보겠습니다.

```

package com.cybb.main;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{
	
	
	private MySystem mySystem;
	
	@Autowired
	public void setMySystem(MySystem mySystem) {
		this.mySystem = mySystem;
	}
	
	
	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		
		mySystem.MysystemReturnMsg();

	}
}


```
이제 더 이상 ApplicationContext 라는 Ioc 컨테이너는 불러오지 않고 오롯이 @Autowired 하나만 가지고 Ioc 컨테이너에 갔다와서 MySystem 타입의 Bean 을 불러와서 값을 집어넣습니다 
그럼 실행을 해보면 

```
MySystemInfo Bean 생성
[2m2023-03-13 14:32:37.225[0;39m [32m INFO[0;39m [35m14004[0;39m [2m---[0;39m [2m[           main][0;39m [36mo.s.b.w.embedded.tomcat.TomcatWebServer [0;39m [2m:[0;39m Tomcat started on port(s): 8080 (http) with context path ''
[2m2023-03-13 14:32:37.239[0;39m [32m INFO[0;39m [35m14004[0;39m [2m---[0;39m [2m[           main][0;39m [36mcom.cybb.main.SpringRestartApplication  [0;39m [2m:[0;39m Started SpringRestartApplication in 2.843 seconds (JVM running for 4.274)
시스템2번정상입니다.

```
런타임시 MySystemInfo Bean 생성 이 되고 그다음 Runner 메서드가 자동으로 콜백함수로 작동해서 MysystemReturnMsg 가 호출되는것을 볼 수 있습니다 우리는 이제 Ioc 컨테이너를 코드에 적어주지 않고도 애노테이션으로만 Ioc 컨테이너를 불러서 Bean 을 주입하는 방식으로 개발을 할것입니다 이런 방식이 현업에서 가장많이 이용되는 방식임으로 반드시 알고 있어야 할 내용입니다 

물론 이제 굳이 생성자 또는 setter 에 @Autowired 를 사용할 필요 없이 

```
@Autowired
private MySystem mySystem;

```
이렇게 필드에 바로 입력을 해도 Spring Ioc 는 바로 해당 타입의 Bean 을 찾아서 주입을 해주게 됩니다.














