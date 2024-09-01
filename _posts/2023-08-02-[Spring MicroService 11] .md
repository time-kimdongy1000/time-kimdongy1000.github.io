---
title: Spring MicroService 11 Spring MicroService EUREKA
author: kimdongy1000
date: 2023-08-02 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

## Eureka 정의 
오늘 부터는 마이크로 서비스의 유레카 서비스에 대해서 공부를 해볼것이다 유레카는 서비스의 등록 및 발견하는 도구입니다, 마이크로서비스들의 정보를 Registry에 등록하고 로드밸런싱을 제공하는 미들웨어서버이다

## Eureka 가 필요한이유 
밑에 사진을 한번보자 

![1](https://github.com/user-attachments/assets/a2928fe6-2684-41b9-ba93-495f0ef1ea52) 전통적인 모놀리스식 방식이다 하나의 클라이언트가 동일한 ip 에 다른 port 로 무한정 요청을 넣을 수 있는 구조이다 이런 구조에서는 중간에 로드벨런싱을 해주는 장치가 없기 때문에 서버 과부화 문제가 발생할 수 있다 (다만 전통적인 모놀리스식 서버들 앞에 L4 장치를 두어서 아키텍쳐적으로 로드벨런싱을 하고 있습니다 이 사진은 극단적 사진입니다)

유레카서버는 특별한 L4장치 없이 자신이 직접 로드밸런싱을 진행할 수 있습니다 그 외에도 자신의 아래에 등록된 서비스들의 클라이언트의 상태를 파악해서 오류가 발생한 것들은 덜어내어 그쪽으로 요청이 가지않게 하는 기능 또한 존재합니다 

## 유레카서비스의 역활 

1. 서비스 등록
  마이크로서비스 인스턴스는 유레카 서버에 자신의 메타데이터(호스트, 포트, 상태 정보 등)를 등록합니다.

2. 비동기 헬스 체크
  서비스 인스턴스는 유레카 서버와 정기적으로 통신하여 자신이 여전히 활성 상태임을 알립니다.

3. 서비스 발견 
  다른 마이크로서비스가 특정 서비스와 통신할 필요가 있을 때, 유레카 서버에 쿼리하여 해당 서비스를 찾습니다

4. 로드 밸런싱 
  클라이언트는 유레카 서버로부터 받은 서비스 인스턴스 목록을 사용해 부하를 분산시킬 수 있습니다.


## 유레카 서버의 구축 
그럼 한번 구축을 해보자 나의 아키텍쳐는 계속해서 config 서버를 계속해서 사용해서 유레카서버를 포함한 모든 인스턴스를 통제할것이다 

## 전체소스
`https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main/enureka-init?ref_type=heads`

소스는 이곳에 담아두었습니다 도커 컴포즈를 포함한 Docker 파일도 각각에 들어 있으니 참고바라겠습니다 

## 유레카 설정 
config 서버를 보다보면 eureka 에 대한 설정이 있는데 이에 대해서만 짧게 정리를 하고 지나가면

```

eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone : "${EUREKA_SERVER_URL:http://eurekaserver:8070/eureka/}"
  instance:
    hostname: localhost

```
1. fetch-registry - 로컬에 등록정보를 캐싱하지 않고 일정시간마다 다시 클라이언트의 상태와 메타데이터를 읽어옵니다 true 로 해두면 캐싱을 한다는 뜻입니다 이때는 유레카서버가 일시적으로 응답하지 않을때에는 로컬에 있는 정보를 클라이언트에 전달해줍니다 

2. register-with-eureka true 로 두면 해당 인스턴스를 유레카 서버에 등록을 할것인지 여부입니다 

3. defaultZone 유레카 서비스의 위치를 명시할 수 있습니다 

## 도커 기동 
파일에 있는 docker-compose 파일을 기동을 해보자 잠시뒤 localhost:8070 유레카 서버에 접속을 하게 되면 유레카서버 대시보드가 나오게 됩니다 

![1](https://github.com/user-attachments/assets/c739ea12-a84c-481a-bbc8-634db2c1d2de)

이렇게 말입니다 다만 서버가 올라오자마자 들어가면 유레카 서버는 들어갈 수 있지만 클라이언트는 아직 검색이 안될 수 있습니다 
인스턴스의 등록은 Instances currently registered with Eureka 에 들어와 있으면 현재 올바르게 인스턴스가 등록이 된것입니다

## 유레카 REST API 
```
http://localhost:8070/eureka/apps/SPRING-CLOUD-EUREKA-CLIENT1

```

이렇게 주소를 입력해서 인스턴스명을 입력하면 

```
{
    "application": {
        "name": "SPRING-CLOUD-EUREKA-CLIENT1",
        "instance": [
            {
                "instanceId": "3d10806619ab:spring-cloud-eureka-client1:9000",
                "hostName": "localhost",
                "app": "SPRING-CLOUD-EUREKA-CLIENT1",
                "ipAddr": "172.21.0.4",
                "status": "UP",
                "overriddenStatus": "UNKNOWN",
                "port": {
                    "$": 9000,
                    "@enabled": "true"
                },
                "securePort": {
                    "$": 443,
                    "@enabled": "false"
                },
                "countryId": 1,
                "dataCenterInfo": {
                    "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
                    "name": "MyOwn"
                },
                "leaseInfo": {
                    "renewalIntervalInSecs": 30,
                    "durationInSecs": 90,
                    "registrationTimestamp": 1724551179208,
                    "lastRenewalTimestamp": 1724551539248,
                    "evictionTimestamp": 0,
                    "serviceUpTimestamp": 1724551179208
                },
                "metadata": {
                    "management.port": "9000"
                },
                "homePageUrl": "http://localhost:9000/",
                "statusPageUrl": "http://localhost:9000/actuator/info",
                "healthCheckUrl": "http://localhost:9000/actuator/health",
                "vipAddress": "spring-cloud-eureka-client1",
                "secureVipAddress": "spring-cloud-eureka-client1",
                "isCoordinatingDiscoveryServer": "false",
                "lastUpdatedTimestamp": "1724551179209",
                "lastDirtyTimestamp": "1724551179178",
                "actionType": "ADDED"
            }
        ]
    }
}

```
이렇게 인스턴스의 상태와 정보를 받아볼 수 있습니다 오늘은 이렇게 유레카서버를 먼저 구축하는것 까지 진행을 하겠습니다 
















