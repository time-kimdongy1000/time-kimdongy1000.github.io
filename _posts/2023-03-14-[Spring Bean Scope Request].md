---
title: Spring Ioc 와 Bean Scope Request
author: kimdongy1000
date: 2023-03-14 13:39
categories: [Back-end, Spring - Core]
tags: [ Bean Scope Request]
math: true
mermaid: true
---

## Bean 의 생성전략 
Request 생성전략은 HTTP 요청 한건당 새로운 빈 인스턴스를 생성하는 스코프입니다 즉 각각의 HTTP 요청이 들어올떄마다 새로운 bean 이 생성이 되며 해당요청이 완료되면 
이 Bean 은 소멸이 됩니다 이때 Bean 의 유지시간은 HTTP 요청이 끝날때 까지 값이 유지가 됩니다 이때는 IoC 와 서블릿컨테이너의 동기화가 이루어져서 
서블릿컨테이너에서 요청이 완료가 되어서 응답이 나갔으면 IoC 컨테이너에 요청을 해서 해당 bean 을 삭제하게 됩니다 

## MySingletonBean
```
@Component
@RequestScope
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
참고로 scope 는 request 가 먹지 않아서 RequestScope 로 대체를 하겠습니다 그럼 이 빈은 HTTP 요청이 올때마다 새로운 bean 을 생성하게 됩니다 

```
@RestController
public class MessageController {
	
	@Autowired
	private MySingletonBean mySingletonBean;
	
	
	@GetMapping("/message1")
	public String message1() {
		
		mySingletonBean.setMessage("message1 mySingletonBean 생성");
		
		try {
			Thread.sleep(10000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		return mySingletonBean.getMessage();
	}
	
	@GetMapping("/message2")
	public String message2() {
		
		mySingletonBean.setMessage("message2 mySingletonBean 생성");
		
		try {
			Thread.sleep(5000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		return mySingletonBean.getMessage();
	}

}

```
그리고 HTTP 요청이 필요함으로 핸들러 2개를 만들고 각 핸들러에 서로 다른 값을 집어넣고 요청순서를 /message2 -> /messag1 순서대로 요청을 하게 됩니다 그러면 일반적인 싱글톤의 결과는 
후에 요청한 message1 요청의 `mySingletonBean.setMessage("message1 mySingletonBean 생성");` 값을 가져오게 됩니다만 Request 스코프를 쓰게 되면 HTTP 요청간의 독립적인 빈이 생성됨으로 이때는 후에 요청한 messag1 값을 가지고 오는것이 아니아 원래 요청한 값을 가져오게 됩니다 

## 리퀘스트의 생성전략의 장점 , 단점 

장점 
1. 요청마다 독립적인 상태 
    프로토타입처럼 새로운 HTTP 요청이 들어올떄마다 새로운 Bean 인스턴스가 생성되기 때문에 각 요청은 독립적인 상태를 가질 수 있습니다 

2. HTTP 요청간 동일한 데이터 유지
    동일한 HTTP 요청에 한해서는 독립적인 값을 계속해서 공유할 수 있습니다 

단점 
1. 메모리 누수 
    HTTP 요청이 끝나면 자동으로 소멸되긴 하지만 그래도 상태변화가 필요 없는 bean 까지 Request 로 생성할 경우 메모리 누수 및 성능저하의 원인이 될 수 있습니다

2. 요청범위 밖의 상태 공유 불가능
    동일한 HTTP 요청일때만 값이 공유되지만 그 요청바깥에 있는 상태에는 관여 또는 공유 할 수 없습니다 

3. 상태관리의 어려움 
    웹어플리케이션은 수많은 요청에 의해서 움직이게 됩니다 이때 이 값이 동일한 HTTP 요청에 한해서 값이 독립적으로 공유가 되어야 하는지 아니면 전역적으로 공유가 되어야 하는지 
    판단을 하고 설계를 진행을 해야 합니다 그래서 복잡한 어플리케이션일 수도록 더욱더 세밀한 bean 생성전략을 사용해야 합니다

오늘시간까지 spring bean 이 생성될 전략에 대해서 공부를 해보았습니다 대표적 3가지만 알아본것이고 이 외에도 session , websocket 다양한 방식이 있습니다 이에 대해서는 다음에 한번 다루어 보는것으로 하겠습니다 