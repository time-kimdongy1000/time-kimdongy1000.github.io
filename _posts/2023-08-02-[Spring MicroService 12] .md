---
title: Spring MicroService 12 Spring MicroService EUREKA - Service discovery
author: kimdongy1000
date: 2023-08-02 11:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  EUREKA]
math: true
mermaid: true
---

## 서비스 디스커버리란 
에를 들어서 유레카서버 안에 A와 B 라는 인스턴스가 있다고 가정을 해보자 A라는 서비스에 B의 어떤 특정한 서비스를 이용하고 싶을때가 있을 수 있다 이때 A서비스는 B의 물리적인 위치를 알 수 없다 그렇기에 유레카서버에게 물어본다 그러면 유레카 서버는 A가 원하는 B의 요청정보를 알려주고 A는 그 요청정보 가지고 자신이 원하는 서비스를 만들 수 있을것이다 

## 전체 소스
https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-discovery?ref_type=heads

이 소스는 총 4개의 프로젝트가 있습니다 컨피그 서버 , 유레카 서버 , 2개의 서로 다른 서비스 인스턴스가 존재하고 유레카 서버는 이를 감지해서 이들의 상태를 레지스트리에 등록을 해둡니다 

## 시나리오 핵심
이 시나리오의 핵심은 Client1 에서 유저를 생성하고 Client2 에서 해당 유저의 출퇴근 시간을 입력하면 Client1 에서 Client2 에 직접 RestApi 를 통해서 데이터를 가져오는 방식을 말합니다 

## eureka - clinet FindService

```
@Component
public class FindService {

    @Autowired
    private DiscoveryClient discoveryClient;

    public String getInstancesUri(@PathVariable String serviceId) {
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
        return  instances.get(0).getUri().toString();
    }
}

```
이 FFindService 를 통해서 이제 유레카서버에 등록된 특정한 서비스의 인스턴스를 찾아오고 해당하는 url 을 가져올것입니다 

## DiscoveryClient 
주된 역활은 유레카 서버에 등록된 모든 인스턴스를 검색하거나 찾을 수 있도록 지원하는 객체입니다 지금같은 소스를 보게 되면 인스턴스 아이디를 가지고 유레카서버에 등록된 인스턴스 정보를 가져오는
로직입니다

## cureka - client EmpContoller 
```

@Autowired
private EmpRepository empRepository;

@Autowired
private FindService findService;

@GetMapping("/read_workTime/{empCode}")
public ResponseEntity<WorkTimeDto> read_workTime(@PathVariable String empCode)
{

    try{

        String service_url = findService.getInstancesUri("spring-cloud-eureka-client2");

        RestTemplate restTemplate = new RestTemplate();

        String request_url = String.format("%s/read_workTime/%s" , service_url , empCode);

        System.out.println("======================================================");
        System.out.println(" request_url : " + request_url);
        System.out.println("======================================================");


        ResponseEntity<WorkTimeDto> restExchange = restTemplate.exchange(
                request_url,
                HttpMethod.GET,
                null ,
                WorkTimeDto.class

        );

        return new ResponseEntity<>(restExchange.getBody() , restExchange.getStatusCode());
    }catch(Exception e){
        throw new RuntimeException(e);
    }
}

```

예를 들어 다음과 같은 소스가 있습니다 이 소스는 시나리오상 등록된 userCode 의 출퇴근 시간을 가져오는 로직입니다 이때 FindService 의 getInstancesUri 를 활용해서 유레카 서버에 등록된 인스턴스의 정보중 url 을 가져와서 Rest 통신을 해서 결과값을 가져오는 방식입니다 이렇게 하면 로그창은 다음과 같이 보이게 될것입니다 

# 결과 로그
```

2024-08-25 20:13:20 ======================================================
2024-08-25 20:13:20  request_url : http://172.21.0.3:9001/read_workTime/2c95808491893cef0191893d376e0000
2024-08-25 20:13:20 ======================================================

```

즉 DiscoveryClient 를 통해서 서비스를 찾아서 통신하는 것을 볼 수 있습니다

## 의문사항 
그럼 질문이 있을 수 있습니다 이런 번잡한 일을 할 이유가 없어보입니다 왜햐하면 모놀리스 서버 구성으로 하면 각각의 Client1 , Client2 의 서버 와 포트 정보를 알 수 있을 것입니다 이는 독립적인 서비스이기 때문이죠 굳이 이 인스턴스들을 유레카서버에 등록해서 서비스끼리 통신할 이유가 없는것 처럼 보입니다 일을 굳이 하나 더 늘리는것처럼 보일것입니다 하지만 마이크로서비스의 핵심은 무엇이냐면 서비스 인스턴스들은 서로의 IP 와 PORT 를 모른다 즉 물리적인 위치를 모르는것에서 부터 출발합니다 

## Client2 번 PORT 은닉 

```
server:
  port: 0
eureka:
  instance:
    preferIpAddress : true
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone : "${EUREKA_SERVER_URL:http://eurekaserver:8070/eureka/}"
      #defaultZone : "${EUREKA_SERVER_URL:http://localhost:8070/eureka/}"
```

config 서버의 Client2 의 port 를 0으로 만들겠습니다 이렇게 하면 이제 클라이언트 2번은 바인딩될 수 있는 무수히 많은 포트중에 랜덤으로 포트가 부여되고 바인딩됩니다 만약 이런 상황이라면 단순 Rest 통신으로는 2번서버의 위치를 알 수 없어 모놀리스 방식으로는 Rest 통신이 불가능하지만 우리는 등록된 유레카 서버로 인해서 동일한 코드로 호출이 가능합니다
그리고 로그를 보면 이렇게 됩니다 

```

2024-09-01 12:08:32 ======================================================
2024-09-01 12:08:32  request_url : http://172.18.0.5:40577/read_workTime/2c95808591ab76350191ab770bdc0000
2024-09-01 12:08:32 ======================================================

```
이렇게 Random 한 port 가 부여되어서 외부에서는 전혀 접근을 할 수 없게 됩니다 지금은 도커 안에 있는 브릿지 네트워크를 써서 IP 는 동일하지만 독립된 서버를 사용시 이 부분 마저도 알 수 없게 처리할 수 있습니다 그렇지만 유레카서버에 등록이 되어 있다면 물리적인 IP 와 Port 를 모르더라도 해당 서비스를 호출 할 수 있습니다


## 아키텍쳐
![1](https://github.com/user-attachments/assets/bd9a61e8-545b-4b8d-9213-d7b3ab2fd8b4)

이제 아키텍처는 이렇게 될것입니다 클라이언트는 이제 공개되어 있는 서버 하나를 알고 있습니다 그리고 이 공개된 서버가 공개되어 있지 않은 서버들과 통신하기 위해서 중간에 유레카서버에서 이들의 적절한 정보를 주고 받게 도와줍니다 물론 서버를 알고 난 뒤에서 공개된 서버와 각각의 인스턴스들은 직접적인 통신이 가능합니다 단 이때 클라이언트에게 숨겨진 서버를 알리지 않기 위함입니다

## 정리
유레카서버 및 디스커버리클라이언트를 쓰는것을 정리를 해보자

1. 유레카서버가 시작이 되면 등록할려고 하는 인스턴스를 자동으로 등록 및 제거하게 됩니다

2. 지난시간에 유레카서비스에 헬스체크를 통해서 인스턴스의 상태를 파악하고 해당 인스턴스를 제거 (배우진 않음)

3. 서비스의 유연한 확장 IP 와 PORT 가 어디에 종속이 되지 않으니 여러 인스턴스가 동적으로 들어서도 문제가 없습니다 모든 인스턴스는 유레카서버에 등록이 되고 유레카서버가 통신을 할 수 있게 도와줍니다 













