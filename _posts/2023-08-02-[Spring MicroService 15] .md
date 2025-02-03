---
title: Spring MicroService 15 Spring MicroService 클라이언트 회복성 및 Resilience4j - fallBack 처리
author: kimdongy1000
date: 2023-08-03 11:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Resilience4j]
math: true
mermaid: true
---

지난시간에 Resilience4j 서킷브레이커에 대해서 알아보았습니다 이번에는 fallBack 처리에 대해서 알아보겠습니다 fallBack 은 중재자로 서비스가 실패할때 해당 api  가로채서 다른 대안을 취할 수 있는 것입니다 

## 전체소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-Resilence4j-fallback?ref_type=heads>


## fallbackMethod 패턴 구현
```

private ResponseEntity<EmpDto>  fallback_read_emp(String empName , Throwable e)
{
    return new ResponseEntity<>(new EmpDto(UUID.randomUUID().toString() , "Service is not available.") , HttpStatus.OK) ;
}

@GetMapping("/read_emp/{empName}")
@CircuitBreaker(name = "read_emp" , fallbackMethod = "fallback_read_emp")
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
소스는 대게 간단합니다 서킷브레이커 안에서 사용할 fallbackMethod 명을 정의해주고 그에 따라서 fallbackMethod 메서드를 구현해주면됩니다 

## 주의
fallbackMethod 를 구현할때는 다음의 경우를 지켜야 합니다

1. 같은 class 에 위치를 해야 합니다 

2. return 타입이 일치해야 합니다 

3. 매개변수가 동일해야 하며 반드시 Throwable e 를 포함해야 합니다 

## FallBack 패턴의 의의
지금은 Fallback 패턴에서 사실상 더미 데이터 return 하면서 그냥 단순하게 끝내는 방식으로 마쳤습니다 다만 실제 클라우드 서비스에는 empName 을 사용해서 사용자 정보를 가져오는 api 가 여럿있기 때문에 실제 FallBack 패턴은 이런 더미데이터를 넣고 return 하는 것이 아니라 비슷한 서비스를 할 수 있는 api 로 연결하는 것이 목적입니다 

## 서킷브레이커 영향
FallBack 패턴을 사용한다고 할지라도 서킷브레이커의 상태는 계속해서 기록이 되고 있는 중입니다 만약 회로차단기 상태가 OPEN 이라면 바로 FallBack 메서드를 호출하는 것입니다 
즉 FallBack 패턴은 2가지 상황에서 발생되는 것입니다 

1. api 를 정상적으로 처리하지 못할때 

2. 회로차단기가 OPEN 상태 또는 HALF OPEN 상태일때 