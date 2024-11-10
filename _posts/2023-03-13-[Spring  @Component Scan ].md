---
title: Spring Ioc @ComponentScan
author: kimdongy1000
date: 2023-03-13 12:00
categories: [Back-end, Spring - Core]
tags: [IoC , '@ComponentScan']
math: true
mermaid: true
---

## @ComponentScan
사실 앞에서 런타임 시 자동으로 bean 을 찾아서 등록해 준다고 했지만 실제로는 @ComponentScan 어노테이션 기반으로 자동으로 등록이 된다 이 어노테이션은 범위 안에 있는 패키지들을 스캔해서 이들 중에서 bean으로 등록될만한 것들을 찾아서 bean 을 만들고 IoC 컨테이너에 넣는 역할을 하게 됩니다 즉 우리가 앞에서 자동으로 자연스럽게 bean 을 불러오는 방식에는
이와 같은 어노테이션이 있기 때문이다 그럼 다시 소스로 와보자 우리 패키지 아무리 보더라도 @ComponentScan 찾아볼 수가 없다 사실 이는 숨겨져 있는데

## @SpringBootApplication
```

@SpringBootApplication
public class SpringRestart2Application implements ApplicationRunner{


}
```
boot 환경에서는 이 어노테이션이 핵심이다 이것으로 인해서 이 애플리케이션이 boot 인지 아닌지 판별을 하게 되는데 

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

}

```

이 어노테이션 안으로 들어가게 되면 그제야 보이는 것이 하나가 있다 바로 @ComponentScan이다 이 @ComponentScan 을 기준으로 런타임 시 애플리케이션의 모든 bean 이 될만한 것들을 탐색하게 됩니다 그래서 기본적인 beanScan 범위는 SpringBootApplication 어노테이션이 달려있는 파일로부터 하위가 된다

그럼 잠깐 파일 위치를 보자

-project 
	-src
		-main
			-java
				-com
					-cybb
						-main
							-SpringRestart2Application.java 
							-ClassRommA.java
							-Student.java 
						-main2
							-Student2.java	

이렇게 되어 있다 즉 SpringRestart2Application.java  파일을 기준으로 동일 레벨 또는 하위 레벨은 전부 대상이 됩니다 그렇기에 지금처럼 main2 - Student2.java는 bean 대상이 아닙니다

```
@SpringBootApplication
public class SpringRestart2Application implements ApplicationRunner{

	@Autowired
	private ApplicationContext applicationContext;

	public static void main(String[] args) {
		SpringApplication.run(SpringRestart2Application.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		

		Student2 student2 = applicationContext.getBean("student2" , Student2.class);
		System.out.print(student2.hashCode());		
	}
}
```
예를 들어서 이와 같이 작성을 하게 되면 bean 을 찾을 수 없게 됩니다 참고로 ApplicationContext 즉 IoC 컨테이너도 외부 주입으로 사용할 수 있습니다 이때는 어떤 에러가 발생하냐면

```
Description:
A component required a bean named 'student2' that could not be found.


Action:
Consider defining a bean named 'student2' in your configuration.

```
student2 bean 이름으로 된 컴포넌트를 찾을 수 없습니다 이렇게 나오게 됩니다 즉 bean 범위가 아닐 때에는 Ioc 컨테이너는 스캔을 하지도 않고 bean 을 만들지도 않게 됩니다
자 그럼 이대로 끝이냐 그렇지 않습니다 사실 spring 은 bean의 범위를 설정을 할 수 있습니다

## 스캔범위 지정 
```
@SpringBootApplication(scanBasePackages = {"com.cybb.*"})
public class SpringRestart2Application implements ApplicationRunner{

	@Autowired
	private ApplicationContext applicationContext;

	public static void main(String[] args) {
		SpringApplication.run(SpringRestart2Application.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		

		Student2 student2 = applicationContext.getBean("student2" , Student2.class);
		System.out.print(student2.hashCode());		
	}
}
```

이때는 상단에 scanBasePackages 을 사용함으로써 bean의 범위를 재지정 할 수 있습니다 이때는 com.cybb 아래에 있음으로 이 아래에 있는 모든 범위로 지정을 하게 되면
이제 spring IoC는 기동 시 이 범위 하단에 있는 모든 bean 을 스캔하고 bean 이 될만한 것들을 넣고 기동을 하게 됩니다
