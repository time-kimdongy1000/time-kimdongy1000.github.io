---
title: Spring MicroService 13 Spring MicroService EUREKA - RestTemplate , Feign
author: kimdongy1000
date: 2023-08-02 12:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

지난시간엔 서비스 디스커버리를 통해서 알 수 없는 인스턴스의 통신을 시도해 보았다 이번시간에는 유레카서버가 지원하는 2가지 서비스 찾기 RestTemplate , Feign 에 대해서 알아보겠습니다 

## 전체소스
https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-restTemplate?ref_type=heads

## RestTemplate - LoadBalanced
이번에는 다른 방식으로 유레카서버를 이용해서 인스턴스를 찾아보겠습니다 이번에 추가된 클라이언트가 있습니다 Client3 번인데 2번 클라이언트와 동일하게 만들어졌습니다 그래서 Docker 로 기동을 하게 되면 

![1](https://github.com/user-attachments/assets/391dfd09-70c1-4734-bbdf-bc46ed0301c5)

기동을 하게 되면 유레카서버에 이렇게 Client2 인스턴스가 2개 잡히게 되지만 이 둘은 엄연히 다른 독립적인 인스턴스입니다 

## Clinet3 - read_workTime 

```

@GetMapping("/read_workTime/{empCode}")
public ResponseEntity<WorkTimeDto> read_workTime(@PathVariable String empCode)
{
    try{

        System.out.println("=====================================================================");
        System.out.println("This Instance is Client - Service3");
        System.out.println("=====================================================================");

        Optional<WorkTimeEntity> optionalWorkTimeEntity = workTimeRepository.findByEmpCode(empCode);
        WorkTimeDto workTimeDto = null;

        if(optionalWorkTimeEntity.isPresent()){

            WorkTimeEntity workTimeEntity = optionalWorkTimeEntity.get();

            workTimeDto = WorkTimeDto.builder()
                    .workTimeSeq(workTimeEntity.getWorkTimeSeq())
                    .empCode(workTimeEntity.getEmpCode())
                    .gwt(workTimeEntity.getGTW())
                    .lw(workTimeEntity.getLW())
                    .setTime(workTimeEntity.getWork_year() + "-" + workTimeEntity.getWork_month() + "-" + workTimeEntity.getWork_day() + " " +workTimeEntity.getWork_localTime())
                    .build();
        }

        return new ResponseEntity<>(workTimeDto , HttpStatus.OK);
    }catch(Exception e){
        throw new RuntimeException(e);
    }
}

```
어떤 인스턴스에서 read_workTime 호출했는지 알 수 있게 2번과 3번 각각 구분할 수 있는 로그를 남기겠습니다 

## Clinet1 - SpringCloudEnureKaClientApplication

```
@SpringBootApplication
@EnableEurekaClient
public class SpringCloudEnureKaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEnureKaClientApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}

```
RestTemplate 에 Bean 을 사용하고 해당 Bean 에 LoadBalanced 어노테이션을 넣게 되면 이는 유레카서버 전용으로 유레카 서버가 이 restTemplate 를 사용한 통신에 대해서 적절한 로드벨런싱을 해주게 됩니다 단 이 객체로 외부api 를 호출 할 수 없습니다 내부 유레카 서버 한정해서만 사용할 수 있고 외부 서버 호출시에는 new 로 새로운 객체를 생성해주면됩니다 

```

@Autowired
private RestTemplate restTemplate;

@GetMapping("/read_workTime/{empCode}")
public ResponseEntity<WorkTimeDto> read_workTime(@PathVariable String empCode)
{

    try{
        ResponseEntity<WorkTimeDto> restExchange = restTemplate.exchange(
                "http://SPRING-CLOUD-EUREKA-CLIENT2/read_workTime/{empCode}",
                HttpMethod.GET,
                null ,
                WorkTimeDto.class,
                empCode
        );

        return new ResponseEntity<>(restExchange.getBody() , restExchange.getStatusCode());
    }catch(Exception e){
        throw new RuntimeException(e);
    }
}
```
이제 Client1 에서 앞에 디스커버리 클라이언트 쓰는것처럼 유레카를 통해서 서비스 호출을 해보도록 하겠습니다 이때 중요한것은 api 주소 전체를 적는 것입니다
즉 api 는 이런 모양입니다 `<유레카서비스명>:<api 호출주소>` 이렇게 됩니다 이렇게 요청을 하면 현재 clinet2 , client3 은 동일한 주소를 가지고 있으므로 유레카서버가 적절히 이를 로드벨런싱 하게 됩니다 그래서 로그를 보게 되면 


```
eurekaclient3  | =====================================================================
eurekaclient3  | This Instance is Client - Service3
eurekaclient3  | =====================================================================

....

eurekaclient2  | =====================================================================
eurekaclient2  | This Instance is Client - Service2
eurekaclient2  | =====================================================================
```

이렇게 동일한 주소로 호출을 넣을 떄마다 유레카서버가 알아서 로드벨런싱을 시키는것을 볼 수 있습니다 

## 로드벨런싱의 의미
로드벨런싱의 의미는 단 한가지다 서버의 과중한 요청을 더는 것이다 네트워크 장비라면 L4 가 있는 것입니다 클라우드는 적절히 해당 api 와 서버를 분석해서 
과부화 여부를 판단해서 분산 요청을 넣게 됩니다 

## 라운드 로빈
클라우드 기반에서는 기본적으로 알고리짐이 라운드 로빈입니다 라운드 로빈은 요청이 들어오는 순서대로 배분을 하게 됩니다 예를 들어서 a 요청을 A 서버에서 처리 했으면 다음 a 요청은 B 서버에서 처리하게 됩니다 

# 그 외
그 외 방법으로 Random 방식과 Weighted Response Time 방식이 있습니다 전자의 방식은 부하 상관 없이 랜덤하게 요청을 주고 뒤에 방식은 요청을 빨리 처리한 서버에 더 많은 요청을 주는 알고리즘입니다 


## Feign 
앞에서 서비스디스커버리 , 로드벨런서 , 마지막으로 Feign 으로 호출하는 방법으로 끝을 내겠습니다 이는 인터페이스로 동작을 하는 유레카 서버 호출방법입니다 

## maven 

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
메이븐 추가 

## Client1  - SpringCloudEnureKaClientApplication 

```
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class SpringCloudEnureKaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEnureKaClientApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}

```

EnableFeignClients 애노테이션 추가를 해줍니다 

## FeignClient2Service
```
@FeignClient("spring-cloud-eureka-client2")
public interface FeignClient2Service {

    @RequestMapping(method = RequestMethod.GET , value = "/read_workTime/{empCode}" , consumes = "application/json")
    WorkTimeDto readWorkTime(@PathVariable("empCode") String empCode);
}
```
이렇게 해서 하나의 인터페이스를 만들어줍니다 이때 FeignClient 찾을 유레카 서비스 아이디를 입력을 해줍니다 

```

@Autowired
private FeignClient2Service feignClient2Service;

@GetMapping("/read_workTime/{empCode}")
public ResponseEntity<WorkTimeDto> read_workTime(@PathVariable String empCode)
{

    try{

        

        System.out.println("============================================================");
        System.out.println("FeignClient 로 호출");
        System.out.println("============================================================");
        return new ResponseEntity<>(feignClient2Service.readWorkTime(empCode) , HttpStatus.OK);
    }catch(Exception e){
        throw new RuntimeException(e);
    }
}

```
이렇게 호출을 하게 됩니다 그러면 유레카 서버는 아까와 마찬가지로 적절히 사용되지 않은 서비스 클라이언트를 찾아서 값을 반환을 해주게 됩니다 
오늘은 여기 까지 해서 서비스 클라이언트를 만들고 이들을 유레카 서버에 등록을 한뒤 포트를 숨기고 유레카 서버를 통해서 통신하는 방법에 대해서 공부를 해보았습니다.

