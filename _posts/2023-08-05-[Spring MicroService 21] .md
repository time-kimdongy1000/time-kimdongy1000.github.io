---
title: Spring MicroService 21 Spring MicroService Spring-Cloud-GateWay 상관관계 추척 및 AfterFilter
author: kimdongy1000
date: 2023-08-05 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  GateWay]
math: true
mermaid: true
---

우리는 지난 시간에 사전 필터를 공부하면서 모든 분산 인스턴스의 관문 역할을 하는 곳에서 특정 헤더 값이 들어올 때만 해당 모듈의 인스턴스를 연결해 주고 그렇지 않으면 차단해서 끝내는 사전 필터에 대해서 공부를 했습니다 이번 시간은 계속해서 관문의 역할을 하는 Spring-Cloud-Gateway에서 특정 API를 추적하는 과정을 살펴볼 것입니다 이를 상관관계 추적이라고 합니다
그리고 마지막인 AfterFilter에 대해서 알아보겠습니다

## 상관관계 
Spring-Cloud-GateWay에서 만든 모든 상관관계는 하위 서비스 호출에도 전파가 됩니다 이는 해당 API 호출 간 유일 값이 되고 이 유일 값을 이용해서 하위 모든 모듈에서 추적을 할 수 있습니다 

역시나 WebFlux이다 보니 배경지식이 그렇게 많이 없어서 Chat-gpt 도움을 받아서 코드를 작성했습니다 

## 전체적인 그림
![1](https://github.com/user-attachments/assets/f14595a1-359b-4c28-a286-27cf6b1e2861)

이런 로직으로 흘러갈 것입니다

1. GateWay-BeforeFilter에서 전체적으로 전파할 각 추적 토큰(상관관계)을 만들 것입니다

2. 전파된 토큰은 각 모듈의 BeforeFilter - AfterFilter에서 로킹을 하는 작업을 진행

3. GateWay-AfterFilter에서 다시 로킹을 해서 클라이언트로 return 될 것입니다

## 전체소스
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-gateway-tracking-filter?ref_type=heads>

## Spring-cloud-gateway BeforeFilter
```
@Order(Integer.MIN_VALUE)
@Component
public class BeforeFilter implements GlobalFilter {

    private static final Logger logger = LoggerFactory.getLogger(BeforeFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)
    {
        try{


            ObjectMapper objectMapper = new ObjectMapper();
            String object_String = "";
            byte[] result_body_byte = null;

            String authorization = exchange.getRequest().getHeaders().getFirst("Authorization");

            if(authorization == null || !authorization.startsWith("Bearer ")){
                logger.error("인증토큰이 없습니다.");

                Map<String , Object> result_map = new HashMap<>();
                result_map.put("ERROR_MSG" , "Empty_Authorization_Code");
                result_map.put("ERROR_CODE" ,HttpStatus.UNAUTHORIZED);

                object_String = objectMapper.writeValueAsString(result_map);

                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                exchange.getResponse().getHeaders().set("Content-Type" , "application/json");
                result_body_byte = object_String.getBytes();

                return exchange.getResponse().writeWith(Mono.just(exchange.getResponse()
                        .bufferFactory()
                        .wrap(result_body_byte)));
            }

            String bearerToken = authorization.substring(7);
            logger.info("인증토큰은 : {}" , bearerToken);



            /*추적 토큰 (상관관계 ID)*/
            String tracking_token = "X_Tracking_Token" + UUID.randomUUID().toString();

            exchange.mutate().request(
                    exchange.getRequest().mutate()
                            .header("X_Tracking_Token" , tracking_token)
                            .header("auth_token" , bearerToken)
                            .build()

            ).build();


        }catch(Exception e){
            throw new RuntimeException(e);
        }

        return chain.filter(exchange);
    }
}

```
상관관계 토큰을 만들어준 뒤 그것을 하위 모듈로 전파하는 과정입니다

## spring-cloud-enureKa-client TrackingTokenHeaderLocalThread
```
@Component
public class TrackingTokenHeaderLocalThread {

    private static final String X_Tracking_Token = "X-Tracking_Token";
    private String x_tracking_token = new String();

    public String getX_tracking_token() {
        return x_tracking_token;
    }

    public void setX_tracking_token(String x_tracking_token) {
        this.x_tracking_token = x_tracking_token;
    }
}

```
먼저 로컬 스레드를 하나 선언을 해줍니다 이는 전역으로 사용할 것이며 로컬 스레드에 관련한 글은 <https://time-kimdongy1000.github.io/posts/Spring-MVC-ThreadLocal/> 참조 바라며
이곳에서 간단하게 설명을 하자면 하나의 스레드 안에 독립적인 저장소를 가지며 해당 스레드에서만 접근이 가능한 저장소입니다

## spring-cloud-enureKa-client TrackingTokenContextHolder

```
public class TrackingTokenContextHolder {

    private static final ThreadLocal<TrackingTokenHeaderLocalThread> TRACKING_TOKEN_HOLDER = new ThreadLocal<>();

    public static TrackingTokenHeaderLocalThread get_token_context(){

        TrackingTokenHeaderLocalThread localThread = TRACKING_TOKEN_HOLDER.get();
        if(localThread == null){
            localThread = new TrackingTokenHeaderLocalThread();
            TRACKING_TOKEN_HOLDER.set(localThread);

        }
        return TRACKING_TOKEN_HOLDER.get();
    }

    public static void trackingTokenHeaderLocalThread_remove(){
        TRACKING_TOKEN_HOLDER.remove();
    }
}

```
실제 로컬 스레드를 만드는 곳은 이곳입니다 만약 로컬 스레드가 비어 있다면 해당 저장소를 만들고 안에 값을 세팅하고 만약 값이 존재하면 그 값을 return 을 하게 됩니다


## spring-cloud-enureKa-client TrackingTokenFilterBefore
```
@Order(Integer.MIN_VALUE)
@Component
public class TrackingTokenFilterBefore implements Filter {

    private static final Logger log = LoggerFactory.getLogger(TrackingTokenFilterBefore.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException
    {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        String tracking_token = httpServletRequest.getHeader("X_Tracking_Token");
        String auth_token = httpServletRequest.getHeader("auth_token");

        log.info("Before_Filter_X_Tracking_Token : {}" , tracking_token);
        log.info("Before_Filter_auth_token : {}" , auth_token);

        TrackingTokenContextHolder.get_token_context().setX_tracking_token(tracking_token);
        log.info("Before_Filter_LocalThread_Token : {}" , TrackingTokenContextHolder.get_token_context().getX_tracking_token());

        chain.doFilter(request, response);

    }
}

```
TrackingTokenFilterBefore에서는 요청 순서에 의해서 제일 먼저 호출이 되는 filter입니다 이때 들어온 헤더 값을 로킹하고 추적 토큰은 이곳에서 로컬 스레드에 저장을 하게 됩니다
`TrackingTokenContextHolder.get_token_context().setX_tracking_token(tracking_token);`


## spring-cloud-enureKa-client TrackingTokenFilterAfter
```
@Order(Integer.MAX_VALUE)
@Component
public class TrackingTokenFilterAfter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(TrackingTokenFilterAfter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException
    {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String auth_token = httpRequest.getHeader("auth_token");
        String tracking_token = TrackingTokenContextHolder.get_token_context().getX_tracking_token();

        log.info("after_filter_auth_token : {}" , auth_token);
        log.info("after_filter_LocalThread_Token : {}" , tracking_token);

        httpResponse.addHeader("auth_token", auth_token);
        httpResponse.addHeader("X_Tracking_Token", tracking_token);

        TrackingTokenContextHolder.trackingTokenHeaderLocalThread_remove();

        chain.doFilter(request, response);
    }
}

```
그리고 하위 모듈의 AfterFilter에서는 GateWay에 response 할 헤더를 세팅 및 로킹을 하고 이를 return 하게 됩니다 이때 헤더로 넘어오는 값을 바로 세팅하는 것이 아니라 로컬 스레드에 있는 값을 세팅해서 보내주게 됩니다 `String tracking_token = TrackingTokenContextHolder.get_token_context().getX_tracking_token();` ,
`httpResponse.addHeader("X_Tracking_Token", tracking_token);`

## Spring-cloud-gateway AfterFilter

```
@Order(Integer.MAX_VALUE)
@Component
public class AfterToken implements GlobalFilter {

    private final Logger log = LoggerFactory.getLogger(AfterToken.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)
    {

        HttpHeaders headers = exchange.getRequest().getHeaders();
        String auth_token = headers.getFirst("auth_token");
        String tracking_token = headers.getFirst("X_Tracking_Token");



        log.info("auth_token : {}", auth_token);
        log.info("tracking_token : {}", tracking_token);

        //exchange.getResponse().getHeaders().add("auth_token" , auth_token);
        //exchange.getResponse().getHeaders().add("X_Tracking_Token" , tracking_token);

        return chain.filter(exchange);
    }
}


```
끝으로 Gateway에서 AfterFilter는 하위 모듈에서 넘어오는 헤더 값을 한 번 더 로킹을 한 뒤에 이를 다시 바로 client로 return 을 하게 되는 것입니다 이렇게 하면 하나의 상관관계 사이클이 완성이 됩니다

로그를 살펴보겠습니다 

```
인증토큰은 : asdasdwqeweasd123456456
auth_token : asdasdwqeweasd123456456
tracking_token : X_Tracking_Tokenad372a40-029f-4183-ac86-46a42f9adbd4

Before_Filter_X_Tracking_Token : X_Tracking_Tokenad372a40-029f-4183-ac86-46a42f9adbd4
Before_Filter_auth_token : asdasdwqeweasd123456456
Before_Filter_LocalThread_Token : X_Tracking_Tokenad372a40-029f-4183-ac86-46a42f9adbd4

after_filter_auth_token : asdasdwqeweasd123456456
after_filter_LocalThread_Token : X_Tracking_Tokenad372a40-029f-4183-ac86-46a42f9adbd4

```
이렇게 나오게 됩니다 그러면 우리는 이제 하나의 상관관계 ID를 만들고 이것을 추적하는 것을 만들어보았습니다.









