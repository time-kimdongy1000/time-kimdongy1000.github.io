---
title: Spring MicroService 26 Spring MicroService 분산 추적
author: kimdongy1000
date: 2023-08-11 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  sleuth]
math: true
mermaid: true
---

마이크로서비스의 아키텍쳐는 모놀리스의 단일 모듈을 여러 모듈로 분해해서 빌드 배포할 수 있다 이렇게 해서 얻을 수 있는 장점은 유연성 독립성을 얻지만 복잡성이라는 비용도 떠안게 된다 
마이크로서비스는 그 특징상 에러가 발생하면 발생한 위치의 디버깅이 상당히 힘들다 이번시간에는 분산 추적에 관련해서 마이크로서비스는 어떻게 대처할 수 있는지에 대해서 알아보도록 하자 

## 사용기술 

1. 스프링 클라우드 슬루스 - 이는 유입되는 모든 HTTP 요청을 상관관계 ID 라고 알려진 추적 ID 로 측정한다 이 작업을 위해 필터를 추가하고 다른 스프링 컴포넌트와 상호 작용하여 생성된 상관관계 ID 는 모든 시스템 호출에 전달한다 

2. 집킨 여러 서비스 간의 트랜잭션 흐름을 보여주는 오픈 소스 데이터 시각화 도구이다 집킨을 이용하면 트랜잭션 흐름을 볼 수 있고 각 독립적인 모듈마다 성능을 시각적으로 확인할 수 있다 

3. ELK 스택 이는 3개의 오픈 소스 결합인 일레스틱 서치 , 로그스태시 , 키바나를 결합하여 만든 프로그램으로 실시간으로 로그를 분석 검색 시각화 할 수 있다 

## 스프링 클라우드 슬루스
우리는 앞에서 직접 상관관계 ID 를 만들고 그것을 추적하는 필터 (사전필터 , 사후필터) 를 만든것을 기억할 것이다 슬루스는 우리가 직접 상관관계ID 를 추가하지 않아도 스스로 생성해 
삽입하고 보여주는 역활을 하는데 그에 대해서 알아보도록 하자 

## 전체소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project-zipkin1?ref_type=heads>

## 의존성 추가
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```
해당 의존성을 추가하면 아무런 세팅을 하지 않아도 기본적으로 로그가 변경이 됩니다 

```

2024-11-29 10:17:38.349  INFO [gateway,ecd3257da540d23f,ecd3257da540d23f] 10480 --- [nio-7002-exec-3] c.e.m.controller.IndexController         :  requestUrl : http://host.docker.internal:7100/welcome
2024-11-29 10:15:16.718  INFO 26400 --- [binder-health-1] org.apache.kafka.clients.Metadata        : [Consumer clientId=consumer-null-1, groupId=null] Cluster ID: ZNDokCxIRvmpFd6aru2rBg

```
위에가 슬루스가 변경한 로그이고 아래가 기본적으로 사용되는 spring 로그입니다 그럼 슬루스가 이 로그를 어떻게 보내는지에 대해서 알아보겠습니다

## 슬루스 로그
```
gateway - 로깅이 시작되는 어플리케이션이름 
ecd3257da540d23f - 추적ID (사용자 요청에 대한 고유 식별자이며 해당 요청의 모든 서비스 호출에 전달됩니다 )
ecd3257da540d23f - 스팬ID (사용자 요청 내 한 세그먼트에 대한 고유 식별자 여러 서비스간 호출해서 사용자 트랜잭션 내 서비스 호출시 하나의 스팬 ID 가 할당)
마지막 부분은 생략되었지만 집킨 전송여부가 있습니다 이 부분은 다시 나오면 보여드리겠습니다 

```

## EmpClinet -> SavingMoney 
그러면 이번에는 EmpClinet 에서 Emp 를 한명 만들도록 하겠습니다 그러면 이때 자동적으로 분산캐싱이 동작되어서 SavingMoney 의 로그가 찍히게 되는데 
```

2024-11-29 10:53:40.058  INFO [miniProject-EmpClient,3fe966990a7f74c1,3fe966990a7f74c1] 24048 --- [nio-7100-exec-4] c.e.m.event.MessageSourceBean            : Seding message CREATE for empClient emp_id c6a417ef-124d-43fc-8742-b63272921289

2024-11-29 10:53:40.064  INFO [miniProject-SavingMoney,3fe966990a7f74c1,05ed49d5e37859bc] 26456 --- [container-0-C-1] c.e.m.MiniProjectSavingMoneyApplication  : Received Message To EmpClient : CREATE , c6a417ef-124d-43fc-8742-b63272921289
2024-11-29 10:53:40.064  INFO [miniProject-SavingMoney,3fe966990a7f74c1,05ed49d5e37859bc] 26456 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : ===============================================
2024-11-29 10:53:40.064  INFO [miniProject-SavingMoney,3fe966990a7f74c1,05ed49d5e37859bc] 26456 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : saveEmpRedisData : 
2024-11-29 10:53:40.064  INFO [miniProject-SavingMoney,3fe966990a7f74c1,05ed49d5e37859bc] 26456 --- [container-0-C-1] c.e.m.c.common.EmpClientRedisTemplate    : ===============================================

```
위에 첫줄은 miniProject-EmpClient 에서 시작된 로그이고 그 아래 4줄은 miniProject-SavingMoney 에서 발생한 슬루스 로그입니다 아마 공통점이 보일것입니다 
첫번재 로그 먼저 보면 추적ID 3fe966990a7f74c1 로 보입니다 그리고 자기 자신이 호출되는것이기에 스팬ID 는 3fe966990a7f74c1 동일하게 나옵니다 
두번째 로그 3fe966990a7f74c1 추적아이디는 시작점인 miniProject-EmpClient 의 추적ID 와 동일하게 됩니다 그것을 공통으로 써서 이 추적ID 가 어디서 부터 시작하게 되는지 알 수 있게 되는것입니다 


## gateway -> EmpClient -> SavingMoney
한단계 더 추가해서 이번에는 로그를 보자 그런데 조금 이상할것이다 우리는 슬루스를 추가했을떄면 모든 HTTP 요청에 대해서 슬루스 로그가 생기고 추적ID , 스팬ID 가 생긴다고 생각할 수 있지만 실상은 그렇지 않다

```

2024-11-29 11:14:19.898  INFO [gateway,,] 21276 --- [nio-7002-exec-1] c.e.m.controller.EmpController           : ================================================================================
2024-11-29 11:14:19.898  INFO [gateway,,] 21276 --- [nio-7002-exec-1] c.e.m.controller.EmpController           : START CREATE EMP
2024-11-29 11:14:20.007  INFO [gateway,,] 21276 --- [nio-7002-exec-1] c.e.m.controller.EmpController           : END CREATE EMP
2024-11-29 11:14:20.008  INFO [gateway,,] 21276 --- [nio-7002-exec-1] c.e.m.controller.EmpController           : ================================================================================

```
gateway 로그를 보면 그렇지 않다는 것이다 중간에 추적ID 와 스팬ID 가 누락되어 있는 모습을 볼 수 있다 

## 슬루스 추적 판별
사실 해당 슬루스의 추적ID , 스팬ID 를 넣는것은 자체적인 판단을 하게 됩니다 슬루스는 다음과 같은 상황에서 추적ID , 스팬ID 를 넣게 됩니다 

1. 자동 생성 - spring Cloud Sleuth는 HTTP 요청, 메시지 큐, 내부 서비스 호출 등 다양한 이벤트에 대해 자동으로 추적 ID와 스팬 ID를 생성하고 이를 로그에 포함시킵니다.
              이떄 말하는 자동생성은 우리가 인위적으로 넣은 log.info 같은 소스가 아닌 IoC 컨테이너로 인해서 생기는 로그를 말합니다 

즉 위에서 

```
2024-11-29 10:53:40.058  INFO [miniProject-EmpClient,3fe966990a7f74c1,3fe966990a7f74c1] 24048 --- [nio-7100-exec-4] c.e.m.event.MessageSourceBean            : Seding message CREATE for empClient emp_id c6a417ef-124d-43fc-8742-b63272921289

```
이 로그는 

```
public interface CustomMessageSource {

    @Output("outputChannel")
    MessageChannel output();
}


@Configuration
@EnableBinding(CustomMessageSource.class)
public class MessageConfig {
}


@Component
public class MessageSourceBean {

    private Logger log = LoggerFactory.getLogger(MessageSourceBean.class);

    private CustomMessageSource source;

    @Autowired
    private MessageSourceBean(CustomMessageSource source) {
        this.source = source;
    }

    public void publicEmpClientChange(StatusEnum statusEnum , String emp_id , EmpDao empDao)
    {
        log.info("Seding message {} for empClient emp_id {}" , statusEnum , emp_id);
        EmpChangeModel empChangeModel = new EmpChangeModel(statusEnum.toString() , emp_id , empDao);

        source.output().send(MessageBuilder.withPayload(empChangeModel).build());
    }
}



```
즉 이때 발생된 로그는 우리가 직접 log.info() 를 통해서 나오는 로그들이 아닌 Bean 의 트리거에 의해서 발생되는 로그 메세지를 말하는 것입니다 

2. 샘플링 
사실 모든 슬루스 로그가 기록되는 것은 아닙니다 기본적인 설정은 총 요청의 10% 정도만 로깅을 하고 그 외에는 전부 폐기를 시킵니다 해당 설정에 대한 비율은 yml 파일에서
`spring.sleuth.sampler.probability` 로 조절할 수 있습니다 

3. 에러로그 추적
샘플링설정이 기본 10% 라고 할지라도 에러가 발생할 시 슬루스는 이를 샘플링 비율에서 제외하고 추적을 하게 됩니다 

오늘은 이렇게 슬루스가 무엇인지에 대해서 알아보았습니다 이제 집킨을 이용해서 해당 로그들을 저장하고 시각화 하는 것들에 대해서 알아볼것입니다 




