---
title: Spring Ioc 와 Bean @Qualifier
author: kimdongy1000
date: 2023-03-14 09:30
categories: [Back-end, Spring - Core]
tags: [ Qualifier ]
math: true
mermaid: true
---

## @Qualifier 
우리는 지난시간에 Bean 의 Primary 설정을 통해서 여러개의 동일한 타입의 Bean 이 있다고 할지라도 우선적으로 Bean 을 주입할 수 있는 애노테이션 
@Primary 로 선언을 했다 @Primary 의 위치는 @Bean 을 정의할때 사용함으로 보통 @Bean , @Component 하고 같이 쓰이는 경우가 많지만 
이번에 배워볼 @Qualifier 는 Bean 을 주입받는 위치에서 어떤 Bean 을 주입받을지 결정한다 아래의 코드를 보자 

```

package com.cybb.main;

public interface MySystemInfo {
	public String mySystemInfo (String msg);	
}

```


```

package com.cybb.main;

import org.springframework.stereotype.Component;

@Component(value = "Info1")
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

@Component(value = "Info2")
public class Info2 implements MySystemInfo {

	@Override
	public String mySystemInfo(String msg) {
		return msg + "2번이 호출되었습니다";
	}
}

```



```

package com.cybb.main;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{
	
	@Autowired
	@Qualifier(value = "Info2")
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

소스만보고도 알아차리겠지만 우리가 앞에서 사용한 애노테이션은 이름을 붙일 수 있다 xml 에서 id 로 구분할 수 있는것을 여기서는 value 라는 값으로 고유값을 설정할 수 있게 된다 `@Component(value = "Info1")` `@Component(value = "Info2")` 이렇게 설정을 함으로써 이 bean 은 Info1 의 이름과 Info2 의 이름을 가지게 되는것이다 
그리고 메인을 보자 

```
@Autowired
@Qualifier(value = "Info2")
private MySystemInfo info;

```

메인을 보면 @Qualifier 를 Info2 로 선언을 했다 이때는 주입받을 시점에 이 Info2의 Bean 을 주입받겠다고 하는것이고 이의 결과는 

```
@Component 를 테스트 중입니다2번이 호출되었습니다

```

이렇게 나오게 된다 즉 Bean 을 생성할때 우선순위를 결정하는 @Primary , Bean 을 주입할때 이름을 통해서 주입하는 @Qualifier 가 각각 다른 상황에서 쓰이니 
적절할때 사용토록 하자 

지금은 @component 에서만 사용했지만 실제로 @Bean 에서도 사용할 수 있다.