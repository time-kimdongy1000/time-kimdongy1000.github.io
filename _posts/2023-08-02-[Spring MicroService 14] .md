---
title: Spring MicroService 14 Spring MicroService 클라이언트 회복성 및 Resilience4j - CircuitBreaker
author: kimdongy1000
date: 2023-08-03 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Resilience4j]
math: true
mermaid: true
---

지난시간까지는 EUREKA 서버를 이용해서 유레카 서버에 등록된 인스턴스끼리 다양하게 통신을 하면서 왜 유레카 서버가 필요한지 알아보았습니다 이번시간에는 클라이언트회복성 
Resilience4j 에 대해서 알아보겠습니다 

## Resilience4j
Resilience4j는 Netflix Hystrix로부터 영감을 받은 함수형 프로그래밍(functional programming)으로 설계된, 경량의 내결함성(fault tolerance) 라이브러리입니다. Resilience4j는 Circuit Breaker 이외에 다른 핵심 모듈인 Bulkhead, RateLimiter, Retry, TimeLimiter, Cache을 제공하고 있습니다.

## Circuit Breaker 
사전적인 의미는 주가의 등락폭이 갑자기 심해질 경우 시장에 미칠 엄청난 충격을 고려하여 주식 매매를 일시적으로 정지시키는 것을 말합니다 그럼 클라이언트 회복성에 빗대어 말해보면 인스턴스 A 에 있는 api a 가 성능이 느려질 경우 Resilience4j 는 a api 의 문제로 인해서 A 인스턴스는 물론 분산 인스턴스 전체가 문제가 되는것을 막기 위해 잠시 a api 를 호출할 수 없게 만듭니다

## 회로차단기 
쉽게 말해서 집에 있는 두꺼집을 생각하면됩니다 두꺼비 집은 집안의 엄청난 전력 과열로 인해서 문제가 생길것을 알고 스스로 두꺼비집을 내려 전력을 차단하고 더 큰 위험을 방지하게끔 설계되어 있습니다 

## Circuit Breaker 원리

![1](https://github.com/user-attachments/assets/f51f53e7-fa76-44e9-aaed-bbeb3d0f72fb)

서킷브레이커는 총 3가지의 상태를 가집니다 

1. Closed - 평소의 api 상태입니다

2. Open - Closed 상태에서 너무 많은 호출 실패 또는 느려짐 같은 성능 에러가 발생하면 Resilience4j 가 자동으로 api 상태를 Open 상태로 만들어서 더 이상 호출하지   못하게 합니다 

3. half Open - Open 상태에서 일정 시간이 흐른뒤 api 상태를 파악해서 다시 정상상태이면 Closed 아니라면 Open 으로 상태를 변경하는 중간 상태입니다

## 원형버퍼
![2](https://github.com/user-attachments/assets/15db1ea1-8cd4-40bf-a7b0-da0f4668905c)


api 가 실패할때마다 이런 원형링에 실패와 성공을 기록하게 됩니다 이 원형에서 실패율이 특정 임계점 이상이 된다면 api 를 차단하게 됩니다 



## 전체 소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-Resilence4j?ref_type=heads>

전체소스는 참고 바랍니다 


## config-server - spring-cloud-enureka-client1-dev
```

management.endpoints.web.exposure.include: '*'
management.endpoint.health.show-details: always
management.health.circuitbreakers.enabled: true

resilience4j:
  circuitbreaker:
    instances:
      read_emp:
        registerHealthIndicator: true     #서킷 브레이커 활성화시 Actuator 의 헬스체크 엔드포인트에 등록할지 여부
        ringBufferSizeInClosedState: 5    #회로 차단기가 닫힌 상태일때 실패 및 성공 요청을 기록하는 원형 버퍼의 크기를 설정합니다
        ringBufferSizeInHalfOpenState: 3  #회로 차단기가 반열림 상태일때 허용된 테스트의 요청의 수를 기록하는 원형 버퍼의 크기를 지정합니다 이때는 시스템이 회복되었는지 판단하는 여부로 확인됩니다
        waitDurationInOpenState: 10s      #회로 차단기가 열림 상태일때 반열림 상태로 전환하는 시간이며 이 시간동안은 모든 api 를 차단합니다
        failureRateThreshold: 50          #회로 차단기가 닫힌 상태일때 열림 상태로 전환하기 위한 임계값을 설정할 수 있습니다

```
먼저 config 서버에 간단한 설정을 보겠습니다 상단에 있는 management 설정은 다양한 helath 를 파악하기 위한 설정입니다 

두번째는 resilience4j 설정을 보면 instances 아래에 특정 인스턴스만 설정할 수 있습니다 그리고 그외는 각각의 주석을 참고 바랍니니다 그리고 

## spring-cloud-enureka-client1 
```
private void sleep(){

    try{

        Thread.sleep(1000);
        throw new TimeoutException("에러가 발생했습니다.");
    }catch(Exception e){
        log.error(e.getMessage());
        throw new RuntimeException(e);
    }
}

private boolean result_true_false(){

    Random random = new Random();

    int rnd_int = random.nextInt(100) + 1;

    return rnd_int % 2 != 0;
}

@GetMapping("/read_emp/{empName}")
@CircuitBreaker(name = "read_emp")
public ResponseEntity<EmpDto> read_emp(@PathVariable("empName") String empName) throws Exception
{
    if(result_true_false()) sleep();

    Optional<EmpEntity> optional_empEntity = empRepository.findByEmpName(empName);

    EmpDto resultEmpDto = null;
    if(optional_empEntity.isPresent()){
        resultEmpDto = new EmpDto(optional_empEntity.get().getEmpCode(), optional_empEntity.get().getEmpName());
    }


    return new ResponseEntity<>(resultEmpDto , HttpStatus.OK);
}


```
3개의 메서드를 준비했습니다 핵심은 read_emp 메서드이며 이곳에서는 `@CircuitBreaker(name = "read_emp")` 앞에서 정의한 circuitbreaker instance read_emp 가 해당됩니다 

## ControllerHandlerAdvise

```
@RestControllerAdvice
public class ControllerHandlerAdvise {

    @ExceptionHandler(RuntimeException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity<?> runtimeHandleException(Exception e) {

        Map<String , Object> resultMsg = new HashMap<>();
        resultMsg.put("errorMsg" , e.getMessage());

        return new ResponseEntity<>(resultMsg ,  HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(CallNotPermittedException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity<?> callNotPermittedHandleException(Exception e) {

        Map<String , Object> resultMsg = new HashMap<>();
        resultMsg.put("errorMsg" , "당분간 핸들러를 호출할 수 없습니다.");

        return new ResponseEntity<>(resultMsg ,  HttpStatus.INTERNAL_SERVER_ERROR);
    }
}


```
잠깐 ControllerHandlerAdvise 설명을 하면 에러가 발생하면 특정 핸들러의 설정으로 인해서 에러메세지를 미리 정해진 규칙으로 만들어져서 return 하게 됩니다 
즉 평소에러라면 `runtimeHandleException` 에러가 발생할것이고 만약 서킷브레이커로 인해서 api 가 open 상태이면 `callNotPermittedHandleException` 호출이 될것입니다 

## resultMsg
```
{
    "errorMsg": "java.util.concurrent.TimeoutException: 에러가 발생했습니다."
}

{
    "errorMsg": "당분간 핸들러를 호출할 수 없습니다."
}

```
메세지는 총 3개가 나올것입니다 성공메세지 , 에러가 발생한 일반 에러메세지 , 서킷브레이커로 인해서 회로가 차단된 후 나오는 메세지 이렇게 나오게 됩니다 


## Health check - OPEN
```
"status": "UP",
"components": {
    "circuitBreakers": {
        "status": "UNKNOWN",
        "details": {
            "read_emp": {
                "status": "CIRCUIT_OPEN",
                "details": {
                    "failureRate": "60.0%",
                    "failureRateThreshold": "50.0%",
                    "slowCallRate": "0.0%",
                    "slowCallRateThreshold": "100.0%",
                    "bufferedCalls": 5,
                    "slowCalls": 0,
                    "slowFailedCalls": 0,
                    "failedCalls": 3,
                    "notPermittedCalls": 1,
                    "state": "OPEN"
                }
            }
        }
    },
}

```
api `http://localhost:9000/actuator/health` 로 실행을 하게 되면 다음과 같은 read_emp 의 현재 api 상태를 볼 수 있습니다 지금은 테스트 때문에 state 가 open 인것을 볼 수 있습니다
open 에서는 특정 시간이 흐른후 waitDurationInOpenState 시간에 따라 설정한 시간이 흐르면 half open 상태로 변경이 됩니다 

## Health check - HALF_OPEN
```
"components": {
        "circuitBreakers": {
            "status": "UNKNOWN",
            "details": {
                "read_emp": {
                    "status": "CIRCUIT_HALF_OPEN",
                    "details": {
                        "failureRate": "-1.0%",
                        "failureRateThreshold": "50.0%",
                        "slowCallRate": "-1.0%",
                        "slowCallRateThreshold": "100.0%",
                        "bufferedCalls": 1,
                        "slowCalls": 0,
                        "slowFailedCalls": 0,
                        "failedCalls": 0,
                        "notPermittedCalls": 0,
                        "state": "HALF_OPEN"
                    }
                }
            }
        },
    }

```

## Health check - CLOSED
```
 "components": {
        "circuitBreakers": {
            "status": "UP",
            "details": {
                "read_emp": {
                    "status": "UP",
                    "details": {
                        "failureRate": "-1.0%",
                        "failureRateThreshold": "50.0%",
                        "slowCallRate": "-1.0%",
                        "slowCallRateThreshold": "100.0%",
                        "bufferedCalls": 0,
                        "slowCalls": 0,
                        "slowFailedCalls": 0,
                        "failedCalls": 0,
                        "notPermittedCalls": 0,
                        "state": "CLOSED"
                    }
                }
            }
        },
    }

```
그리고 half - open 에서 에서 closed 로 갈떄는 ringBufferSizeInHalfOpenState 의 설정에 따라서 성공률을 기록하고 그 기록에 따라서 OPEN 으로 갈지 CLOSED 로 갈지 결정이 됩니다 