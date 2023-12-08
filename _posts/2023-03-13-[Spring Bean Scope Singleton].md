---
title: Spring Ioc 와 Bean Scope Singleton
author: kimdongy1000
date: 2023-03-13 14:00
categories: [Back-end, Spring - Core]
tags: [ Bean Scope Singleton]
math: true
mermaid: true
---

## Bean 의 생성전략 
기본적으로 Spring 의 Bean 의 생성전략은 싱글톤입니다 

## 싱글톤 (Singleton Strategy)
우리는 앞에서 계속해서 Bean 으로 생성된것들은 IoC 컨테이너가 관리한다는 말을 계속 보고 있다 이때 이 IoC 는 이 bean 의 전략을 보고 어떻게 bean 을 만들지 전략을 만들게 되는데 
이 기본적인 전략은 싱글톤입니다 싱글톤은 단 한개의 bean 만 생성이 되고 이 bean 인스턴스를 컨테이너가 유지하고 이를 여러부분에서 공유하고 사용합니다 

그러면 여러군대에서 공유되고 있다는 것을 소스로 한번 보게 되면 아래와 같습니다 

## MySingletonBean
```
@Component
public class MySingletonBean {
	
	private String message;
	
	public void setMessage(String message) {
		this.message = message;
	}
	
	public String getMessage() {
		return message;
	}
}


```
이 MySingletonBean 는 bean 으로 만들어서 message 를 세팅해서 showMessage 를 호출하게 되면 저장된 메세지를 보여주는 아주 간단한 bean 입니다 

## Message1
```
@Component
public class Message1 {

	@Autowired
	private MySingletonBean mySingletonBean;

	public void showMessage() {
		System.out.print("Message1 call : " + this.mySingletonBean.getMessage() + "\n");
	}

	public void setmessage(String message) {

		mySingletonBean.setMessage(message);
	}
}

```

## Message2
```
@Component
public class Message2 {

	@Autowired
	private MySingletonBean mySingletonBean;

	public void showMessage() {
		System.out.print("Message2 call : " + this.mySingletonBean.getMessage() + "\n");
	}

	public void setmessage(String message) {

		mySingletonBean.setMessage(message);
	}
}

```

## Message3 
```
@Component
public class Message3 {

	@Autowired
	private MySingletonBean mySingletonBean;

	public void showMessage() {
		System.out.print("Message3 call : " + this.mySingletonBean.getMessage() + "\n");
	}

	public void setmessage(String message) {

		mySingletonBean.setMessage(message);
	}
}

```

지금보면 Message1 , Message2 , Message3 은 전부 MySingletonBean을 의존받고 있고 showMessage 함수는 MySingletonBean 저장된 message 를 저장하는 함수이고 
setmessage 는 MySingletonBean 에 접근해서 메세지를 변경하는 것이다 그럼 이것으로 이게 싱글톤하고 무슨 관계인가 할 수 있지만 
다시 한번 싱글톤의 정의를 살펴보면 "싱글톤은 단 한개의 bean 만 생성이 되고 이 bean 인스턴스를 컨테이너가 유지하고 이를 여러부분에서 공유하고 사용합니다"
즉 bean 은 런타임시 단 한번만 생기게 되고 의존하는 모든곳에서 동일한 메모리 주소로 접근을 하게 됩니다 즉 어디 한곳에서 메세지를 변경하면 그 메세지 변경은 
호출하는 모든 Message 에서 변경점이 반영되는것입니다 

```
@SpringBootApplication
public class SpringRestart2Application implements ApplicationRunner{
	
	 
	@Autowired
	private MySingletonBean mySingletonBean;
	
	@Autowired
	private Message1 message1;
	
	@Autowired
	private Message2 message2;
	
	@Autowired
	private Message3 message3;
	
	

	public static void main(String[] args) {
		SpringApplication.run(SpringRestart2Application.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		
		mySingletonBean.setMessage("singletoneBean 세팅");
		
		message1.showMessage();
		message2.showMessage();
		message3.showMessage();
		
		message1.setmessage("Message1 에서 메세지 변경");
		
		message1.showMessage();
		message2.showMessage();
		message3.showMessage();	
	}	
}
```
자 이제 실행을 하게 되면 첫번째 호출할떄 `mySingletonBean.setMessage("singletoneBean 세팅");` 기본메세지를 세팅을 하게 되면 
```
message1.showMessage();
message2.showMessage();
message3.showMessage();

```
이 3개는 동일한 메세지 
```

Message1 call : singletoneBean 세팅
Message2 call : singletoneBean 세팅
Message3 call : singletoneBean 세팅

```

이렇게 나오게 됩니다 즉 호출하는 메서드는 서로 다르더라도 공유하고 있는 메세지는 동일한것을 볼 수 있습니다 마찬가지로 `message1.setmessage("Message1 에서 메세지 변경");`
다음으로 호출을 하게 되면 

```

Message1 call : Message1 에서 메세지 변경
Message2 call : Message1 에서 메세지 변경
Message3 call : Message1 에서 메세지 변경

```
메세지가 동일하게 변경되는점을 볼 수 있습니다 이게 기본적인 spring bean 의 생성전략중 하나인 싱글톤 생성전략입니다 

## 싱글톤 생성전략의 장점 , 단점 

장점 
1. 자원절약 
	지금보면 한번 bean 을 만든뒤에 총 3곳에서 호출했지만 동일한 bean 이 여러번 생성되는것이 아니라 하나의 bean 이 여러번 생성되는것이기 때문에 메모리 절약의 효과가 있습니다 

2. 애플리케이션의 전역 상태관리
	하나의 빈으로 애플리케이션 전체적으로 공유가 됨으로 DB 접속정보같이 한번 입력하면 굳이 변경될 내용이 아닌경우에는 다른곳에서도 똑같은 bean 을 여러번 불러올 수 있습니다 

단점
1. 상태변경시 동기화
	지금 위의 예제처럼 어디에서든 기본적인 message 를 변경할 수 있는 소스에서는 싱글톤 환경에서는 주의가 필요합니다 예를 들어서 `singletoneBean 세팅` 우리는 기본메세지가 이렇게 설정을 했는데 다른곳에서 접근해서 메세지를 수정하게 되면 전역으로 메세지가 수정되기 떄문에 싱글톤 빈의 값이나 , 상태를 변경할때는 주의가 필요합니다 

2. 서버확장에 대한 공유자원문제 
	서버클러스팅 같이 여러서버에서 애플리케이션이 동작하는 경우 싱글톤 빈의 상태 공유 문제가 발생할 수 있습니다 말이 좀어렵다면 다음과 같은 예시를 들어보자 

	가정 
	1. 애플리케이션 A 가 서버 B , 서버 C 에 설치가 되어 있고 A 애플리케이션의 모든 bean 은 싱글톤에 상태를 변경할 수 있는 D Bean 이 존재합니다 
	
	2. E 클라이언트가 B 서버에 접근해서 D Bean 의 상태를 변경합니다 

	3. F 클라이언트가 C 서버에 접근해서 D Bean 의 상태를 조회

이런 가정을 생각한다면 

A 애플리케이션의 D Bean 은 서버 B 서버 C 각각의 메모리에 싱글톤 bean 을 생성하게 됩니다 그래서 E 클라이언트가 변경한 내역이 F 클라이언트가 조회했을땐 서로의 상태를 알 수 없는 환경이기 때문에 서로 다른 상태를 읽게 되는 것입니다 

그래서 이와같은 장점과 단점을 충분히 고려한 어플리케이션 개발이 필요합니다