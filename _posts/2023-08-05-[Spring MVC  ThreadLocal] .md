---
title: Spring MVC ThreadLocal
author: kimdongy1000
date: 2023-04-21 10:10
categories: [Back-end, Spring - MVC]
tags: [ MVC , ThreadLocal ]
math: true
mermaid: true
---

## ThreadLocal
각 스레드가 독립적으로 값을 저장하고 사용할 수 있도록 하는 메커니즘을 제공합니다. 이는 다중 스레드 환경에서 여러 스레드가 같은 객체를 동시에 사용하지 않도록 해주는 중요한 도구입니다.

## SecurityContext
우리는 시큐리티를 하면서 ThreadLocal를 간접적으로 사용했습니다 그 객체는 바로 SecurityContext입니다 이는 시큐리티에서 현재 세션의 인증정보를 담아두는 최종 객체로 이는 ThreadLocal 기반으로 만들어져 있고 각 스레드마다 독립적인 값을 저장하고 있으며 서로 다른 스레드에서는 접근이 불가능합니다

## java 사용법

```
public class ThreadLocalTestJava {

    private static final ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public static void main(String[] args)
    {

        Thread thread1 = new Thread(() -> {
            threadLocal.set("thread1");
            System.out.println("Thread-1: " + threadLocal.get());
        });

        Thread thread2 = new Thread(() -> {
            threadLocal.set("thread2");
            System.out.println("Thread-2: " + threadLocal.get());
        });

        thread1.start();
        thread2.start();
    }
}


```
먼저 기본적인 사용법을 알아보겠습니다 현재 사용할 ThreadLocal 은 String 값을 저장하는 객체로 만들어져 있습니다 이때 static fianl로 구성한 이유는 한번 구성된 값들은 변경할 수 없게 만들었습니다 그리고 static를 쓴 이유는 전역으로 고정된 값을 사용하기 위함입니다

## 결과
```

Thread-1: thread1
Thread-2: thread2

```

결과는 서로 다른 스레드에서 생성된 값들은 전혀 다른 스레드에 영향을 미치지 못하고 있는 모습을 보이고 있습니다 전역으로 사용하고 객체 메모리 값을 공유하는 static 을 사용했음에도 말이죠
이런 이유를 가지는 이유는 ThreadLocal 특징 중 하나입니다

## MVC 사용법
그럼 이번에는 웹에서 사용법을 한번 봅시다 MVC는 모든 요청에 대해서 서로 다른 스레드를 가지게 됩니다 즉 어떤 api A에 대해서 서로 다른 클라이언트가 A에 대한 api를 호출하면 각각의 클라이언트는 동일한 스레드 환경에 있는 게 아니라 서로 다른 스레드에서 동일한 로직을 처리하게 됩니다 즉 MVC 요청 하나하나가 전부 ThreadLocal로 동작을 하게 됩니다


## HttpHeaderLocalThread
```
@Component
public class HttpHeaderLocalThread {

    public static final String AUTH_TOKEN = "AUTH_TOKEN";

    private String authToken = new String();

    public String getAuthToken() {
        return authToken;
    }

    public void setAuthToken(String authToken) {
        this.authToken = authToken;
    }
}


```
ThreadLocal 은 위에서 단순 String으로 넣었지만 지금처럼 어떤 타입으로 정의할 수 있습니다


## HttpHeaderLocalThreadContext
```
public class HttpHeaderLocalThreadContext {

    private static final ThreadLocal<HttpHeaderLocalThread> HTTP_HEADER_CONTEXT = new ThreadLocal<>();

    public static final HttpHeaderLocalThread GET_HEADER_CONTEXT()
    {
        HttpHeaderLocalThread localThread = HTTP_HEADER_CONTEXT.get();
        if(localThread == null){

            localThread = new HttpHeaderLocalThread();
            HTTP_HEADER_CONTEXT.set(localThread);
        }

        return HTTP_HEADER_CONTEXT.get();
    }
}

```
이곳에서 ThreadLocal 정의하고 전역으로 사용할 수 있게 private static final로 정의해서 사용하게 됩니다 이때 get() 메서드 호출 시 null이면 새로운 객체를 만들어서 세팅을 하게 됩니다

## HttpBeforeFilter
```
@Component
@Order(Integer.MIN_VALUE)
public class HttpBeforeFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(HttpBeforeFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException
    {

        HttpServletRequest httpServletRequest = (HttpServletRequest) request;

        String auth_token = httpServletRequest.getHeader("auth_token");

        log.info("auth_token : {}" , auth_token);
        log.info("Before_ThreadLocalValue : {}" , HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().getAuthToken());

        if(!StringUtils.hasText(auth_token)){
            if(!StringUtils.hasText(HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().getAuthToken())){
                auth_token = UUID.randomUUID().toString();
            }


        }

        HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().setAuthToken(auth_token);
        log.info("After_ThreadLocalValue : {}" , HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().getAuthToken());



        chain.doFilter(request, response);
    }
}



```
그리고 Fitler를 정의해서 우선순위를 최상위로 올립니다 `@Order(Integer.MIN_VALUE)` 이때 헤더 값에 auth_token 없으면 사용하려는 ThreadLocal에 값을 꺼내서 넣게 되는데 이때도 없으면 새로 만들어서 넣게 됩니다 `HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().setAuthToken(auth_token);`를 이용해서 사용하려는 로컬 스레드 값에 넣게 됩니다 

## HttpAfterFilter
```
@Component
@Order(Integer.MAX_VALUE)
public class HttpAfterFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(HttpAfterFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException
    {

        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String auth_token = HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().getAuthToken();
        log.info("auth_token : {}" , auth_token);

        httpResponse.addHeader("auth_token", auth_token);


        chain.doFilter(request, response);
    }
}

```
그리고 Fitler를 정의해서 우선순위를 최하위로 내립니다 `@Order(Integer.MAX_VALUE)` 마지막으로 Response 넘겨서 클라이언트에 값을 받게끔 합니다
`HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().setAuthToken(auth_token);`를 이용해서 사용하려는 로컬 스레드 값에 넣게 됩니다

그리고 결과를 보게되면

```

2024-10-05 23:16:33.147  INFO 7784 --- [nio-8080-exec-6] c.e.d.T.HttpBeforeFilter                 : auth_token : null
2024-10-05 23:16:33.147  INFO 7784 --- [nio-8080-exec-6] c.e.d.T.HttpBeforeFilter                 : Before_ThreadLocalValue : 
2024-10-05 23:16:33.147  INFO 7784 --- [nio-8080-exec-6] c.e.d.T.HttpBeforeFilter                 : After_ThreadLocalValue : ead7f7f9-4c75-4587-9e85-1d87564be1e5
2024-10-05 23:16:33.147  INFO 7784 --- [nio-8080-exec-6] c.e.d.T.HttpAfterFilter                  : auth_token : ead7f7f9-4c75-4587-9e85-1d87564be1e5

```

이렇게 나오게 됩니다 즉 여러 번 호출하더라도 서로 다른 스레드에서 동작을 하기 때문에 계속해서 새로운 저장소를 만들어서 값을 저장하고 return 을 하게 됩니다

## 추가부분 
ThreadLocal는 메모리 누수를 발생할 수 있습니다 그렇기에 사용하고 나면 반드시 명시적으로 remove를 사용해서 제거를 해야 합니다

## remove 추가
```
public class HttpHeaderLocalThreadContext {

    private static final ThreadLocal<HttpHeaderLocalThread> HTTP_HEADER_CONTEXT = new ThreadLocal<>();

    public static final HttpHeaderLocalThread GET_HEADER_CONTEXT()
    {
        HttpHeaderLocalThread localThread = HTTP_HEADER_CONTEXT.get();
        if(localThread == null){

            localThread = new HttpHeaderLocalThread();
            HTTP_HEADER_CONTEXT.set(localThread);
        }

        return HTTP_HEADER_CONTEXT.get();

    }

    public static final void REMOVE(){
        HTTP_HEADER_CONTEXT.remove();
    }


}

```
추가 메서드를 통해서 ThreadLocal를 명시적으로 제거하게 합니다

## HttpAfterFilter 수정
```
@Component
@Order(Integer.MAX_VALUE)
public class HttpAfterFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(HttpAfterFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException
    {

        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String auth_token = HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().getAuthToken();
        log.info("Current_ThreadLocalValue : {}" , auth_token);

        httpResponse.addHeader("auth_token", auth_token);

        HttpHeaderLocalThreadContext.REMOVE();
        log.info("Remove_ThreadLocalValue : {}" , HttpHeaderLocalThreadContext.GET_HEADER_CONTEXT().getAuthToken());

        chain.doFilter(request, response);
    }
}

```
`httpResponse.addHeader("auth_token", auth_token);` 할당 이후에 `HttpHeaderLocalThreadContext.REMOVE();` 호출해서 ThreadLocal를 제거합니다

오늘은 ThreadLocal에 대해서 알아보았습니다.