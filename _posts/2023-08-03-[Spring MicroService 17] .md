---
title: Spring MicroService 17 Spring MicroService 클라이언트 회복성 및 Resilience4j - 속도 제한기 
author: kimdongy1000
date: 2023-08-03 12:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Resilience4j]
math: true
mermaid: true
---

지난시간에는 Retry 패턴을 배워보았고 이번시간에는 속도 제한기 패턴을 알보도록 하겠습니다 

## 속도제한기 
속도제한기는 일정시간이내에 처리할 수 있는 요청의 수를 제한함으로써 시스템에 가해지는 부하를 조절 할 수 있습니다 

## 2개의 구현체
SemaphoreBasedRateLimiter AtomicRateLimiter 에 대해서 알아볼것입니다 기본적으로 RateLimiter 의 구현체가 2개가 있습니다 이전 구현체는 
SemaphoreBasedRateLimiter 이지만 버전이 올라가면서 기본 구현체는 AtomicRateLimiter 가 되었습니다 

## SemaphoreBasedRateLimiter
이는 하나의 세마포드에 현재 스레드 허용 수를 저장하도록 구현이 되어 있습니다 이 경우 모든 사용자 스레드는 semaphore.tryAcquire() 메서드를 호출하여 현재 호출할 수 있으면 try 를 반환하고 그렇지 않으면 false 를 반환해서 클라이언트가 호출하지 못하게 막습니다 

## AtomicRateLimiter
이는 원자적 연산과 토큰 버킷을 알고리즘을 사용하여 속도제한을 구현합니다 



## 속도제한기 전체 소스 코드 

<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-Resilence4j-SemaphoreBasedRateLimiter?ref_type=heads>

## spring-config-server spring-cloud-eureka-client1-dev 

```
resilience4j:
  ratelimiter:
    instances:
      create_cmp_ratelimiter:
        limit-for-period: 5        # 사이클동안 호출할 수 있는 회수
        limit-refresh-period: 10s  # 사이클이 가지는 주기
        timeout-duration: 5s       # 다시 호출할 수 있는 사이클을 얻을 수 있게 대기하는 시간
```

먼저 yml 을 세팅을 하겠습니다 이때 이 설정값을 머리에 두고 다음 원리를 설명하겠습니다 

## SemaphoreBasedRateLimiter 원리 

1. 세마포어 초기화 
     limit-for-period 값에 따라서 최초 설정할 호출 최대 회수를 설정합니다 

2. 요청 허용 여부 판단
     요청이 들어오면 세마포어 내부에서 tryAcquire 를 호출 이때 return 값이 true 이면 아직 허용가능한 요청으로 퍼미션을 허용하고 세마포어 값을 1 감소시킵니다  
     그렇게 세마포어 값이 0이 되면 tryAcquire 호출시 false 로 반환이 되며 이때 퍼미션 불허용하며 timeoutDuration 값에 따라서 클라이언트는 대기합니다 

3. 주기적 갱신
   현재 상태가 세마포어 값이 0보다 큰 경우 limit-refresh-period 시간동안 사이클을 회복하게 됩니다 만약 세마포어 값이 0인 경우 timeout-duration 동안 호출하지 못하고 
   세마포어 값이 회복되기를 기다립니다 

이렇게 속도를 제한함으로서 서버의 회복을 기대하고 성능을 저하시키지 않게 합니다 

## AtomicRateLimiter 원리 

1. 토큰 버킷 초기화 
    limit-for-period 값에 따라서 최초 설정할 토큰의 개수를 설정합니다 

2. 요청처리
    요청이 들어올 때마다 현재 토큰 수를 원자적 연산으로 감소시킵니다. 토큰이 남아있으면 요청을 허용하고, 그렇지 않으면 제한됩니다.

3. 토큰 리필 
    토큰을 리필하여 새로운 요청을 허용할 수 있게 합니다.


## SemaphoreBasedRateLimiter vs AtomicRateLimiter

SemaphoreBasedRateLimiter - 간단한 동시성 제어와 , 소규모 트래픽에 용이 , 비동기 지원이 되지 않는 단점이 있습니다 

AtomicRateLimiter - 원자적 변수를 통한 요청 수 제한 , 대규모 트래픽에 용이 , 복잡한 내부구현 하지만 사용자는 쉽게 사용가능함 (프레임워크가 결정) , 비동기 지원

## 원자적 연산 
지금 이 글을 보면 원자라는 말이 엄청나온다 이는 화학에서 원자랑 동일하면 돌턴의 원자설에 나오는 그 원자가 맞고 더 이상 쪼갤 수 없는 물질 프로그램에서는 어떤 작업이나 연산이 더 이상 쪼개지지 않고 중간상태가 존재하지 않는 특정을 말합니다 이는 다중 스레드 환경에서 중요한 로직이며 데이터의 일관성을 유지하는데 매우 중요합니다 

## 원자적 연산의 특징 

1. 불가분성  - 연산이 중단되거나 , 다른 연산과 섞일 수 없습니다 하나의 연산이 시작되면 완료 될때 까지 다른 연산이 개입할 수 없습니다 

2. 일관성 유지 - 여러 스레드가 동시에 접근해도 데이터의 일관성이 유지 됩니다 

3. 경쟁조건 방지 - 다중 스레드가 접근할 때 발생하는 경쟁조건을 방지합니다 

CS 에서 말하는 원자는 다음 JAVA 에서 다시 정리하도록 하겠습니다 



## spring-cloud-enurea-client

```

@PostMapping("/create_emp")
@RateLimiter(name =  "create_cmp_ratelimiter" , fallbackMethod = "fallback_create_emp")
public ResponseEntity<EmpDto> create_emp(@RequestBody EmpDto empDto)
{
    try{

        EmpEntity empEntity = EmpEntity.builder().empCode(UUID.randomUUID().toString()).empName(empDto.getEmpName()).build();
        EmpEntity result_empEntity = empRepository.saveAndFlush(empEntity);


        return new ResponseEntity<>(new EmpDto(result_empEntity.getEmpCode(), result_empEntity.getEmpName()) , HttpStatus.OK);
    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

private ResponseEntity<EmpDto>  fallback_create_emp(EmpDto empDto , Throwable e)
{
    return new ResponseEntity<>(new EmpDto(UUID.randomUUID().toString() , "Rate limit exceeded. Please try again later.") , HttpStatus.OK) ;
}

```

소스는 이렇게 구현이 됩니다 마찬가지로 fallback 메서드를 구현하고 만약 퍼미션 미획득으로 인해서 timeoutDuration 에 걸리면 클라이언트에게 fallback - message 를 return 하게 하고 자신은 그동안 회복을 하게 됩니다 

![1](https://github.com/user-attachments/assets/87866f5c-bd1d-4e28-aabf-4212cf551dca)
![2](https://github.com/user-attachments/assets/9031584c-9344-4939-8b13-9e95f1c6dbc6)

그래서 이 두개의 사진을 보면 처음에는 계속해서 동일한 api 를 날리다가 어느순간 클라이언트가 대기하가다 저 메세지를 볼 수 있게 됩니다 

오늘은 이렇게 클라언트 회복성 패턴에서 속도 제한기에 대해서 알아보았습니다 
