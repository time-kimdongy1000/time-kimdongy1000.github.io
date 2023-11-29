---
title: Spring Ioc 와 Bean Scope Request
author: kimdongy1000
date: 2023-03-14 13:39
categories: [Back-end, Spring - Core]
tags: [ Bean Scope Request]
math: true
mermaid: true
---

## Request 
앞에서 사용한 Singleton , ProtoType 는 기본 스프링 Ioc 단계에서 제공하는 스프링 Bean 정책이다 이번에 배울 Request 는 Web 과 관련이 있는 Bean 의 스코프이다 
Request 스코프는 HTTP 로직과 같이 움직이는것으로 request , response 가 끝이나면 자동으로 소멸된다 요청수에 따라서 Bean 이 생성되기는 하지만 
이 Bean 들은 Prototype 하고는 다르게 공유되지 않고 자신의 http 요청에 대해서만 Bean 을 공유한다 그런 예시를 한번 만들어보자 참고로 web 에서 동작하기에 
클라이언트는 post - man 을 활용할것이다 

## 소스코드
```
@RestController
public class InterRestController {

    @Autowired
    private RequestBeanSystem requestBeanSystem;

    @Autowired
    private SystemService systemService;

    @GetMapping("/")
    public SystemModel index(@ModelAttribute SystemModel systemModel)
    {

        try{

            requestBeanSystem.setSystemName(systemModel.getSystemName());
            requestBeanSystem.setSystemValue(systemModel.getSystemValue());
            System.out.println(requestBeanSystem);

            Thread.sleep(10000);

            systemService.systemOutInfo();

            return null;

        }catch(Exception e){
            throw new RuntimeException(e);
        }
    }
}


```

## SystemModel.java
```
public class SystemModel {

    private String systemName;
    private int systemValue;

    public String getSystemName() {
        return systemName;
    }

    public void setSystemName(String systemName) {
        this.systemName = systemName;
    }

    public int getSystemValue() {
        return systemValue;
    }

    public void setSystemValue(int systemValue) {
        this.systemValue = systemValue;
    }
}


```

## SystemService.service
```
@Component
public class SystemService {

    @Autowired
    private RequestBeanSystem requestBeanSystem;

    public void systemOutInfo(){

        System.out.println(requestBeanSystem);
        System.out.println(requestBeanSystem.getSystemName());
        System.out.println(requestBeanSystem.getSystemValue());
    }
}

```
RequestBeanSystem.java
```

package com.example.demo.config;

public class RequestBeanSystem {

    private String systemName;
    private int systemValue;

    public String getSystemName() {
        return systemName;
    }

    public void setSystemName(String systemName) {
        this.systemName = systemName;
    }

    public int getSystemValue() {
        return systemValue;
    }

    public void setSystemValue(int systemValue) {
        this.systemValue = systemValue;
    }
}
```


```
@Configuration
public class WebConfig {

    @Bean
    @RequestScope
    public RequestBeanSystem requestBeanSystem(){

        return new RequestBeanSystem();
    }
}

```

일단 지금 파트는 mvc 파트가 아니기 때문에 간단하게 설명을 할려고 한다 일단 우리는 postman 에서 8080 으로 요청을 넣을것이다 그럼 이 요칭이 SystemModel 타고 올것인데 
이때 요청은 파라미터를 보낼것이다 SystemModel 에 있는 두개의 필드 값으 보내고 
이때 requestBeanSystem 주입하는데 WebConfig 파일을 보면 Bean 의 생성정책이 request 정책이다 즉 http 요청이 들어올때마다 생성이 되고 http 요청이 끝나면 
자동으로 Ioc 컨테이너에서 제거 된다 그런데 중간에 값을 setter 하고 10초를 주는 이유는 우리는 요청을 두번 연속줄것인데 요청 2번 연속을 줄 수 없으니 10초 대기를 하고 
있다가 진행을 할것이다 이때 우리는 요청을 두번 넣을 것이다 그럼 소스 설명은 대략적으로 간것이고 결과를 보자


```
com.example.demo.config.RequestBeanSystem@71c34c0e
com.example.demo.config.RequestBeanSystem@50ef480
com.example.demo.config.RequestBeanSystem@71c34c0e
삼성전자
1000
com.example.demo.config.RequestBeanSystem@50ef480
LG전자
2000

```

일단 요청이 2번 들어 갔는데 Bean 이 두개 생긴것을 확인했다 그리고 그 각각의 Bean 이 우리는 Service 로 값을 넘겨주지도 않았지만 계속 가지고 있다가 자신의 요청에서 가지고 있었던 값들을 넘겨주었다 이것이 requestScope 의 특징이다 

그렇다면 이런 Bean 생성 정책을 다시 Singleton 으로 변경을 해보다 아래처럼 결과가 나올것이다 
## 다시 Singleton
```
com.example.demo.config.RequestBeanSystem@6ace8861
com.example.demo.config.RequestBeanSystem@6ace8861
com.example.demo.config.RequestBeanSystem@6ace8861
LG전자
2000
com.example.demo.config.RequestBeanSystem@6ace8861
LG전자
2000

```

RequestScope 는 자신이 http 요청때 가지고 있는 값을 계속해서 가지고 있었지만 Sigleton 은 그 10 초 기다리는 사이에 다른 값이 들어와서 값을 덮어 써버린 결과가 나온것이다 그래서 항상 제일 마지막에 요청이 들어오는 값이 남게 되는것이다 그럼 이제 prototype 를 살펴보자 


그럼 prototype 하고의 차이점을 살펴보자

```

com.example.demo.config.RequestBeanSystem@3e645809
com.example.demo.config.RequestBeanSystem@3e645809
com.example.demo.config.RequestBeanSystem@4376b7c4
null
0
com.example.demo.config.RequestBeanSystem@4376b7c4
null
0

```
프로토 타입은 이렇게 나온다 왜냐하면 처음 주입할때 즉 RestController 에서 주입받을때 새로운 Bean 을 주입받는데 이때 하나의 메모리를 가지는 Bean 생겨나게 되고 
다음은 service 에서 주입받을때 새로운 주소값을 가지고 있는 Bean 이 생겨나기 때문에 이는 값전달이 제대로 되고 있지 않은 모습을 보여주고 있는것이다 


## 그럼 웹개발할때는 RequestScope 를 써야 하는가?
아니다 내가 웹개발 5년차인데 이 스코프를 쓰는 사람을 한번도 못보았다 기본적으로 RequestScope 는 http 전달에서 사용하는것이 맞고 실제로 
RequestScopee 는 thread safe 하기 때문에 웹개발할때는 이것을 써도 된다 다만 요청을 할때마다 새로운 인스턴스가 생기는 문제점때문에 
만약 요청이 많이 몰리는 시스템이고 굳이 굳이 값을 전달할때는 Bean 으로 전달하기 보다는 Map 으로 전달하는것이 보는 사람의 입장에서도 좋기 때문에 
그렇게 개발하지는 않아보인다 하지만 아까도 보았듯이 멀티 쓰레드 환경에서는 싱글톤 또한 독이 될 수 있음으로 이럴때에는 RequestScope 가 낫다는 뜻이다 