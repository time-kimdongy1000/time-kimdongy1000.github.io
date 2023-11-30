---
title: Spring Ioc @Component
author: kimdongy1000
date: 2023-03-12 12:00
categories: [Back-end, Spring - Core]
tags: [IoC , '@Bean']
math: true
mermaid: true
---

우리는 앞에서 xml 방식이 아닌 애노테이션 방식으로 설정파일을 만들어 IoC 컨테이너에 Bean 을 주입하는 방시에 대해서 기술을 했습니다 이제는 IoC 컨테이너에 직접 넣지도 않고 
기동시 알아서 읽는 방법에 대해서 기술하겠습니다 

## ApplicationContext 는 없어도 된다 

```

ApplicationContext applicationContext = new AnnotationConfigApplicationContext(BeanConfig.class);
ClassRoomA roomA = applicationContext.getBean(ClassRoomA.class);

```
사실 bean 을 생성하고 불러올때 부터 ApplicationContext 코드는 처음부터 필요 없었습니다 단지 이 코드를 쓴 이유는 우리가 만들어진 Bean 들이 IoC 컨테이너에 들어간다는 것을 보여주기 위함이었습니다 실제로 @Bean 태그를 쓴 이후 부터는 아래와 같이 작성할 수 있습니다 

```
@SpringBootApplication
public class SpringRestart2Application implements ApplicationRunner{
	
	@Autowired
	private ClassRoomA classRoomA;

	public static void main(String[] args) {
		SpringApplication.run(SpringRestart2Application.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		

		classRoomA.classRoomAInStudent();
				
	}
}

```
사실 이렇게만 쓰더라도 우리가 원하는 bean 객체를 계속해서 불러올 수 있습니다 SpringBoot 환경이라서 좀 그렇게 보일 수 있지만 실제로는 

```

@Autowired
private ClassRoomA classRoomA;

```
이 코드만 있으면 IoC 컨테이너에서 ClassRoomA 라는 객체를 가져와서 주입을 받게 됩니다 그러한 역활을 하는 애노테이션은 @Autowired
즉 앞에서 @Configuration 이라는 애노테이션 단 클래스 파일을 만들고 그 아래 @Bean 이라는 정의서를 만들게 되면 이들은 애플리케이션이 기동이 될때 자동으로 IoC 컨테이너로 들어가서 
bean 이 됩니다 그리고 외부에서 필요한곳에 주입을 해줄때에는 @Autowired 를 통해서 주입을 하게 되면 IoC 컨테이너가 적절한 bean 객체를 찾아서 주입을 하게 되고 
우리는 그 주입된 bean 객체를 사용할 수 있게 됩니다 

이와 더불어서 사용할 수 있는 애노테이션은 바로 @Component 입니다 이것도 @Bean 태그와 마찬가지로 Bean 객체를 만들때 사용하지만 사용하는 위치가 조금 다릅니다 

## @Component
```

@Component
public class Student {
	
	private String name;
	
	private int age;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	@Override
	public String toString() {
		return "Student [name=" + name + ", age=" + age + "]";
	}
}

```

```
@Component
public class ClassRoomA {
	
	private Student student;
	
	@Autowired
	public ClassRoomA(Student student) {
		
		student.setName("Time");
		student.setAge(20);
		
		
		this.student = student;
		
	}
	
	public void classRoomAInStudent() {
		
		System.out.print(student.toString());
	}
}
```

자 우리는 다른 방법으로 Bean 을 만들었다 앞에서는 메서드 위에 @Bean 붙이면서 이 메서드는 객체를 반환하니 이를 bean 객체로 만들어주세요 했다면 이제는 Class 상단에 @Component 를 붙여서 이 Class 는 bean 객체로 만들어주세요 하고 있습니다 그리고 이때 ClassRoomA 는 외부에서 Student 객체를 주입을 받아야 함으로 @Autowired 사용했습니다 

우리는 점점 bean 객체를 생성하는 코드가 추상화 되는것을 보고 있습니다 IoC 의 단점중 하나가 학습곡선이 높다는것과 추상화가 너무 크다는 게 단점입니다 처음에는 bean.xml 로 bean 을 정의해서 읽게 시키면서 점점 bean 정의 파일은 없어지고 간단한 애노테이션 몇가지로 bean 을 정의하고 있는 모습을 볼 수 있습니다 

자 그럼 여기서 궁금한점은 우리가 앞에서 IoC 컨테이너를 직접불러서 사용할때에는 어떤 bean 을 가지고 와야 하는지 정확히 표현을 했습니다 
`ClassRoomA roomA = applicationContext.getBean("classRoomA" , ClassRoomA.class);` 이렇게 getBean 이라는 메서드를 사용할때 ClassRoomA.class 의 bean 타입을 가져오라고 했습니다 
다만 지금보면 @Autowired 가 적혀만 있지 위에 처럼 명시하는것도 찾는걸까 

사실 여기엔 이미 IoC 메커니즘이 동작을 하고 있습니다 통상적을는 타입을 보고 찾습니다 그것이 무슨말이냐 

우리는 앞에서 ClassRoomA , Student 타입의 bean 을 정의했습니다 이때 Bean 을 만드는 전략중 하나인 동일한 이름의 Bean 은 만들 수 없다는 것입니다 예를 들어서 우리가 위치가 다른 곳에 동일한 Student 파일을 만든뒤 그것을 Bean 으로 해서 기동하는 순간

```
Annotation-specified bean name 'student' for bean class [com.cybb.main.main1.Student] conflicts with existing, non-compatible bean definition of same name and class [com.cybb.main.Student]

```

이렇게 이미 존재하는 Studen 이름의 bean 이 존재하기 떄문에 이 bean 은 생성할 수 없습니다 그래서 기동하다가 시스템이 멈추게 됩니다 그래서 이름이 동일한 bean 은 만들 수가 없게 됩니다 
이 정책을 근거로 IoC 이는 이름은 유일하고 이제 타입만 살펴보면 되기 때문에 주입할려는 대상의 이름 그리고 타입을 분석해서 가장 적절한 bean 을 주입해주게 됩니다