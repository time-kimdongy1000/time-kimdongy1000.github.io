---
title: Spring MicroService 16 Spring MicroService 클라이언트 회복성 및 Resilience4j - Retry 처리
author: kimdongy1000
date: 2023-08-03 12:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Resilience4j]
math: true
mermaid: true
---

지난시간에 Resilience4j 서킷브레이커에 대해서 알아보았습니다 이번에는 Retry 에 대해서 알아보겠습니다 영어단어만 봐서 아시겠지만 이 패턴은 재시도 패턴입니다 

## Retry 패턴
서비스가 처음 실패했을 때 서비스와 통신을 재시도 하는 역활입니다 즉 네트워크 일시적 요류가 발생했을때 해당 작업을 일정 시간동안 반복해서 재시도 하여 성공할 수 있도록 하는 
패턴입니다 

## Retry 동작 방식

1. 성공 -> 최초 api 호출시 성공하면 동작하지 않습니다

2. 실패 -> api 호출시 실패하면 설정된 재시도 횟수와 대기시간을 바탕으로 해당 작업을 다시 시도합니다 

3. 실패/성공 -> 주어진 대기시간 안에 한번이라도 성공하면 이를 성공처리 하고 성공 Message 를 반환합니다 

4. 실패/실패 -> 주어진 대기시간동안 모두 실패하는 경우 이를 최종 실패 처리하고 Fallback 처리를 합니다 

## 전체소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-Resilence4j-retry?ref_type=heads>

## config-server - spring-cloud-eureka-client1-dev.yml
```
resilience4j:
  circuitbreaker:
    instances:
      read_emp:
        registerHealthIndicator: true     #서킷 브레이커 활성화시 Actuator 의 헬스체크 엔드포인트에 등록할지 여부
        ringBufferSizeInClosedState: 5    #회로 차단기가 닫힌 상태일때 실패 및 성공 요청을 기록하는 원형 버퍼의 크기를 설정합니다
        ringBufferSizeInHalfOpenState: 3  #회로 차단기가 반열림 상태일때 허용된 테스트의 요청의 수를 기록하는 원형 버퍼의 크기를 지정합니다 이때는 시스템이 회복되었는지 판단하는 여부로 확인됩니다
        waitDurationInOpenState: 10s      #회로 차단기가 열림 상태일때 반열림 상태로 전환하는 시간이며 이 시간동안은 모든 api 를 차단합니다
        failureRateThreshold: 50          #회로 차단기가 닫힌 상태일때 열림 상태로 전환하기 위한 임계값을 설정할 수 있습니다
  retry:
    instances:
      read_emp_retry:
        maxRetryAttempts: 1   #재시도 회수
        waitDuration: 10000    #재시도 대기시간
        retry-exceptions:     # 예상되는 에러
          - java.util.concurrent.TimeoutException

```
먼저 사용할 retry 을 정의해줍니다 그리고 여기서 중요한점은 재시도 회수랑 재시도 대기시간입니다 즉 10초동안 1번 재시도 할것인데 이때 한번이라도 성공 또는 전부 실패하면 위에 
WorkFlow 처럼 흘러가는 것이고 예상되는 에러만 체크하게끔 합니다 이때 모든 에러를 다 retry 를 하게 되면 오히려 서버 부하가 발생합니다 에러가 발생하는 것들도 재시도를 하기 떄문이죠
그렇기에 retry-exceptions 을 반드시 정의를 해주는것이 좋습니다 

## 실행 
실행하게 되면 사실 크게 체감이 되지 않습니다 그 이유는 10초 안에 1번을 재시도(언제 실행되는지는 알 수 없음) 그 안에 최종실패 / 최종성공했을때 메세지가 나오게 됩니다 
최종 실패하면 Fallback 메서드가 호출이 되는것이고 그렇지 않으면 성공 메세지가 호출이 되는것입니다 

## 주의점
서킷브레이커와 , Retry 에 Fallback 메세지를 같이 붙여놓았지만 우선순위는 서킷브레이커가 Fallback 메세지 우선순위를 가집니다 

## 서킷브레이커 상호보안관계
Retry 는 일시적인 오류에 대응하며 이때 발생한 오류에 대해서는 서킷브레이커 링버퍼에 기록이 되지 않습니다 링버퍼에 기록이 되는것은 Retry 를 통해서도 최종 실패가 떨어진 경우 
기록이 되는 형태입니다 그렇기 때문에 단순 오류 같은 것은 Retry 로 처리를 해서 성공률을 높이고 만약 일정 실패율을 기록하게 된다면 서킷브레이커가 회로를 열어서 더이상 호출이 되지 않게 막는 형태입니다 이때는 Retry 를 타지 않고 바로 서킷브레이커의 Fallback 으로 메서드가 호출이 됩니다 

