---
title: Spring MicroService 20 Spring MicroService Spring-Cloud-GateWay 사전필터
author: kimdongy1000
date: 2023-08-04 11:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  GateWay]
math: true
mermaid: true
---

우리는 지난시간에 클라우드 게이트를 구성하고 완전히 클라이언트로 부터 숨겨져 있는 인스턴스를 엔드포인트로 호출하는 것을 찾아보았습니다 오늘은 클라우드의 다른 기능인 
사전필터에 대해서 알아보도록 하겠습니다 

## 필터
이는 MVC 나 시큐리티를 하게 되면 지겹도록 듣게 되는 클래스 중에 하나입니다 역활은 정수기 필터처럼 여과하는 것입니다 정수기 필터는 특정 불순물을 걸러서 우리에게 꺠끗한 정수가 도달 할 수 있게 해주는 역활을 합니다 코드안에서 필터 안에도 똑같은 역활을 합니다 특정 요청은 불순물처럼 걸러서 엔드포인트에 도달하지 못하게 막는 역활입니다 

## 그럼 왜 Cloud - GateWay 인가?
사실 Filter 는 그냥 MVC 라면 쉽게 구현할 수 있다 굳이 Cloud-GateWay 에 한정된 내용이 아니다 하지만 우리의 구성도를 다시 한번 살펴보자 

![2](https://github.com/user-attachments/assets/45e9868d-0049-47e6-81b8-c518ffbce3c6)

이 그림을 보자 GateWay 는 클라이언트가 도달할려고 하는 엔드포인트의 중간 포인트다 즉 중간관문 즉 모든 요청은 GateWay 를 통하지 않으면 안되기 때문이다 그렇기에 이 GateWay 에서 다양한 작업을 할 수 있다 그중 하나가 특정 Filter 를 설정해서 엔드포트에 도달할려는 클라이언트의 요청을 조절 할 수 있다 우리는 그중에서 사전필터에 대해서 알아볼것이다 

## 전체코드
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-gateway-before-filter?ref_type=heads>

## 사전필터
![3](https://github.com/user-attachments/assets/7bf48361-9bc5-42a2-819d-8bbeec9abe53)

먼저 그림을 보자 앞부분 Spring-Cloud-GateWay 부문만 떼어낸것입니다 이때 필터를 사용하면 이처럼 수 많은 필터를 설정할 수 있습니다 우리는 그 중에서 제일 먼저 동작하게 할 수 있는 사전필터 (순서 1번) 을 만들어보겠습니다 이 필터에서는 특정 헤더값을 체크해서 값이 없으면 서비스 인스턴스 엔드포인트에 도달하지 못하고 요청을 반환하는 방법으로 작성되었습니다 

## 주의 
Spring - boot 로 넘어오면서 RestApi -> WebFlux 로 웹 프레임워크로 개발이 되었습니다 그렇기에 현재 나오는 코드들은 WebFlux 기반으로 작성이 되었고 저도 익숙지 않기에 GPT 도움을 받았습니다 

## BeforeFilter - Spring-cloud-gateWay
```
@Order(1)
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

      }catch(Exception e){
          throw new RuntimeException(e);
      }

      return chain.filter(exchange);
    }
}


```
당연한 이야기겠지만 필터는 게이트웨이에서 작성이 되어야 합니다 `@Order(1)` 순서는 이렇게 제일 최상으로 잡고 진행을 하면 모든 요청의 첫번째의 filter 가 됩니다 
우리는 특정 헤더의 값 `Authorization` 의 값을 감지해서 그 값이 있으면 서비스 인스턴스로 요청을 넣게 되고 그렇지 않으면 401 에러로 반환을 하게 됩니다 

그래서 POST - man 으로 요청을 넣게 되면 

## 각 요청 결과

![4](https://github.com/user-attachments/assets/57f59742-680c-4968-80d3-a6bd714a563c)

![5](https://github.com/user-attachments/assets/b7653f44-efee-4e4b-ab04-8c22a639e3f6)



이렇게 나눠지게 됩니다 오늘은 spring-cloud-gateway 에서 사전필터를 이용해서 특정 요청을 막는 방어로직을 만들어보았습니다 