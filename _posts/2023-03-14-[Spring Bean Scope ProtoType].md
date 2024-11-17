---
title: Spring Ioc 와 Bean Scope ProtoType
author: kimdongy1000
date: 2023-03-14 13:39
categories: [Back-end, Spring - Core]
tags: [ Bean Scope ProtoType]
math: true
mermaid: true
---

## Bean 생성전략 
Spring 은 bean의 생성 전략 기본은 싱글톤 전략인데 이 중에서도 다른 전략을 선택할 수 있습니다 그중에서 프로토타입 전략을 사용할 수 있습니다

## 프로토타입 (ProtoType Strategy)
프로토타입은 매번 bean 을 요청할 때마다 새로운 인스턴스를 생성하여 반환합니다 싱글톤하고는 다르게 bean의 생성 전략을 지정을 해주어야 합니다

## MySingletonBean

```
@Component
@Scope(value = "prototype")
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


지난번 Singleton 전략 소스 그대로에 MySingletonBean의 전략을 `@Scope(value = "prototype")`로 지정을 해서 bean 생성 전략을 지정할 수 있습니다 이제 이렇게 실행을 하게 되면 


```

Message1 call : null
Message2 call : null
Message3 call : null
Message1 call : Message1 에서 메세지 변경
Message2 call : null
Message3 call : null

```
아까 싱글톤하고는 완전히 다른 상황이 보이게 됩니다 이게 프로토타입의 특징입니다 매번 새로운 객체를 생성하게 됩니다 그럼 생성은 어떻게 되느냐

```

@Component
public class Message1 {

	@Autowired
	private MySingletonBean mySingletonBean;
}

@Component
public class Message2 {

	@Autowired
	private MySingletonBean mySingletonBean;
}

@Component
public class Message3 {

	@Autowired
	private MySingletonBean mySingletonBean;
}

```
우리는 각각 Message1 , Message2 , Message3 할 때 MySingletonBean를 @Autowired 하고 있습니다 이때마다 새로운 메모리 주소의 MySingletonBean 생성하게 됩니다

```

Message1 call : null
Message2 call : null
Message3 call : null

```
이때는 Message1 , Message2 , Message3에서 의존 받은 MySingletonBean에 대한 message 가 없기 때문에 전부 null 이 나오게 됩니다


```

Message1 call : Message1 에서 메세지 변경
Message2 call : null
Message3 call : null

```
그럼 여기서 메시지가 나오게 된 이유는 `message1.setmessage("Message1에서 메시지 변경");` 여기에서 우리는 함수를 호출해서 MySingletonBean 메모리의 message에 문자열을 저장하게 됩니다 그렇기 때문에 이때는 `Message1 call : Message1에서 메시지 변경` 이 나오게 됩니다 이때는 새롭게 생겨나게 된 메모리 주소에 message를 입력해서 데이터가 나오게 됩니다 

## 프로토타입 생성 전략의 장점, 단점

장점
1. 독립적인 상태
싱글톤하고 다르게 프로토타입은 독립적인 상태로 하나의 빈이 다른 곳에 영향을 주지 않습니다

단점
1. 메모리 누수
    만약 상태변화 없는 bean 을 계속해서 생성하게 됨으로 이는 메모리 누수 성능 저하를 불러올 수 있습니다

싱글톤으로 생성된 bean 은 Ioc 컨테이너에 의해서 생성 및 해체가 가능하지만 프로토타입으로 생성된 bean 같은 경우는 bean으로 생성은 되지만 Ioc 컨테이너가 생성 해체를 담당하지는 않습니다 프로토타입의 빈 이 생성이 될 때마다 새로운 인스턴스가 생성되고 해당 인스턴스는 클라이언트의 참조되는 한 유효합니다 그 이후 사용되지 않는 것이 GC에 의해서 감지가 된다면 이는 Ioc 컨테이너에 의해서 없어지는 것이 아니라 GC에 의해서 무차별적으로 없어지게 됩니다        
