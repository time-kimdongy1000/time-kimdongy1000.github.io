---
title: Spring MicroService 24 Spring MicroService 비동기 메세지 처리
author: kimdongy1000
date: 2023-08-09 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Security]
math: true
mermaid: true
---

지난시간까지 KeyClock 를 이용해서 어플리케이션의 보안에 대해서 공부를 해보았습니다 이번시간에는 비동기 메세지 처리 Kafaka 에 대해서 알아보겠습니다 먼저 비동기 메세지 처리란 무엇인지 부터 알아보겠습니다 

## 비동기 메세지 처리
서로다른 애플리케이션이나 서비스간에 메세지를 비동기적으로 전달하여 작업을 수행하는 방식입입니다 비동기라는 말에서 알 수 있다 싶히 요청 - 응답 형태가 아닌 메세지를 생산하는 쪽에서는 
받는 사람의 상태와 관련이 없이 메세지를 송신할 수 있습니다 마찬가지로 메세지를 소비하는 입장에서는 송신자와 상관 없이 메세지를 수신할 수 있습니다 

## 클라우드 서비스와의 관계
비동기 메세지처리가 필요한 이유가 무엇일까 예를 들어서 지난시간에 했던 미니 프로젝트에 대해서 알아보자 empClient 는 계속해서 사원을 생성 , 읽기 , 삭제 , 수정 하는 로직이 담겨 있고 
SavingMoney 에는 EmpClient 의 데이터를 읽어서 랜덤하게 상여금을 주는 시스템을 만들었습니다 만약 empClient 가 불안정해지면 어떻게 될까요? 분산 시스템 특정상 하나의 시스템이 불안하더라도 전체적인 시스템의 shutdown 을 유발하지 않습니다 하지만 어떤 프로세스는 서비스간 강한 결합을 가지게 되는데 이럴 수록 분산시템의 효능이 사라지게 됩니다 그래서 우리는 
이 시스템을 비동기 메세지 처리로 느슨하게 만드는 작업을 진행할것입니다 

## 어떻게??
지금의 결합이 왜 강한 결합인지 설명을 드리겠습니다 일단 모듈은 물리적으로 떨어져 있지만 SavingMoney 의 api 는 반드시 empClient 의 서비스를 호출해야 합니다 반대로 gateWay 가 호출해야 할 수도 있습니다 이처럼 강한 결합은 해당 모듈의 장애 발생시 똑같은 오류가 발생됨으로 주기적으로 해당 모듈의 상태를 체크할려고 할때 이 비동기 메세지 처리를 사용합니다

## 종류 
1. 동기식 메세지 처리 - 이는 단순히 요청 - 응답관의 관계의 메세지 처리로 분산시스템에서는 효용력이 떨어집니다 상태를 체크해야 하는 모듈이 불안정할때 사용하기 힘든 메세지 처리 기법입니다 

2. 비동기식 메세지 처리 - 이는 상대편 모듈에 상관 없이 메세지를 발행하고 그 메세지를 카프카 같은 큐에 담아놓고 수신자가 받을 수 있을떄 언제든지 받을 수 있게 대기를 하게 됩니다 

우리는 이번시간에 카프카와 spring - cloud 서비스를 활용해서 먼저 메세지를 담아서 보는 역활을 하고 그 다음시간 이 둘의 모듈을 느슨하게 만들도록 하겠습니다 

## 전체소스코드
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project-messageQ?ref_type=heads>

## docker 
도커에는 앞으로 2개의 새로운 모듈이 들어옵니다 

```

zookeeper:
    image: bitnami/zookeeper:latest
    environment:
        ALLOW_ANONYMOUS_LOGIN : yes
    ports:
        - "2181:2181" 
    container_name: zookeeper      

kafka:
    image: bitnami/kafka:latest
    environment:
        KAFKA_BROKER_ID: 1
        KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
        KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
        KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
        KAFKA_AUTO_CREATE_TOPICS_ENABLE : false

```
카프카와 주키퍼가 들어오고 카프카가 비동기 메세지처리의 핵심모듈이고 zookeeper 가 카프카를 보조해서 활용이 됩니다 

## config-server empClient-dev.yml

```
spring:
  cloud:
    stream:
      bindings:
        outputChannel:
          destination: empClient-topic
          content-type: application/json

      kafka:
        binder:
          brokers: kafka
          zkNodes: kafka

```
메세지 발행자는 empClient 가 할것입니다 emp 데이터의 변경을 메세지에 담고 해당 메세지를 구독하는 서비스들에게 일괄적으로 던지는 역활을 하게 됩니다 

## empClient pom.xml
```

<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-stream -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-stream-kafka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>


```
pom.xml 에 의존성을 추가해줍니다 

## empClient CustomMessageSource.java 
```
public interface CustomMessageSource {

    @Output("outputChannel")
    MessageChannel output();
}

```
이전버전까지는 MessageChannel 자동으로 지원이 되었었는데 3버전떄부터는 원하는 스스로 만들어서 주입을 해주어야 합니다 우리는 메세지 발송을 채널을 `outputChannel` 하겠습니다 
이는 위에서 yml 을 보면 `bindings.outputChannel` 과 일치해야 합니다 


## empClient MessageConfig
```
@Configuration
@EnableBinding(CustomMessageSource.class)
public class MessageConfig {
}


```
위에서 만든 MessageSource 를 bean 설정파일로 만들어서 주입할 수 있게 만들어줍니다 

## empClient MessageSourceBean
```
@Component
public class MessageSourceBean {

    private Logger log = LoggerFactory.getLogger(MessageSourceBean.class);

    private CustomMessageSource source;

    @Autowired
    private MessageSourceBean(CustomMessageSource source) {
        this.source = source;
    }

    public void publicEmpClientChange(StatusEnum statusEnum , String emp_id)
    {
        log.info("Seding message {} for empClient emp_id {}" , statusEnum , emp_id);
        EmpChangeModel empChangeModel = new EmpChangeModel(statusEnum.toString() , emp_id);

        source.output().send(MessageBuilder.withPayload(empChangeModel).build());
    }
}

```

여기서는 이제 메세지를 만들고 내보내는 역활을 하게 되는것입니다 그러면 이 메세지를 카프카의 메세지 큐에 쌓이게 되고 메세지 소비자가 받을 수 있게 카프카가 준비를 하게 됩니다 

## empClient EmpController
```
@RestController
public class EmpController {

    @Autowired
    private EmpRepository empRepository;

    @Autowired
    private MessageSourceBean messageSourceBean;

    private static final Logger log = LoggerFactory.getLogger(EmpController.class);

    @PostMapping("/createEmp")
    public ResponseEntity<EmpDao> createEmp(
            @RequestBody EmpDao empDao
    ) {

        try{

            String emp_id = UUID.randomUUID().toString();
            EmpEntity empEntity = new EmpEntity(emp_id, empDao.getEmp_name() , empDao.getEmp_position(),  empDao.getEmp_phone());

            empRepository.save(empEntity);

            Optional<EmpEntity> find_empEntity_opt = empRepository.findById(emp_id);

            EmpDao resultEmpDao = null;
            if(find_empEntity_opt.isPresent()){
                resultEmpDao = new EmpDao(find_empEntity_opt.get().getEmp_id() ,
                                          find_empEntity_opt.get().getEmp_name() ,
                                          find_empEntity_opt.get().getEmp_position() ,
                                          find_empEntity_opt.get().getEmp_phone());

                messageSourceBean.publicEmpClientChange(StatusEnum.CREATE , resultEmpDao.getEmp_id());
            }

            return new ResponseEntity<>(resultEmpDao , HttpStatus.OK);
        }catch(Exception e){
            log.error(e.toString());
            return new ResponseEntity<>(null , HttpStatus.OK);
        }
    }
}

```
그럼 우리는 emp 데이터가 변환이 생길때 `messageSourceBean.publicEmpClientChange(StatusEnum.CREATE , resultEmpDao.getEmp_id());` 이 코드를 호출해서 empClient 가 카프카에게 메세지를 발송하게 합니다 여기서는 한곳만 표기가 되었지만 전체 소스코드에는 read , update , delete 모두에 표기를 해주었습니다 그럼 보내는 쪽은 이 정도가 되고 

## config-server savingMoneyClient-dev.yml

```

spring:
  cloud:
    stream:
      bindings:
        inputChannel:
          destination: empClient-topic
          content-type: application/json

      kafka:
        binder:
          brokers: kafka
          zkNodes: kafka

```
이제 받는쪽의 카프카 설정을 정의를 해야 합니다 `empClient-topic` 이때 destination 은 생산자와 같은 값이어야 받을 수 있습니다 

## savingMoney pom.xml
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-stream-kafka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>

```
마찬가지로 2개의 의존성을 추가를 해줍니다 

## savingMoney pom.xml

```
public interface CustomSink {

    @Input("inputChannel")
    SubscribableChannel input();
}

```
이전 버전에는 특별한 bean 생성 없이 바로 사용이 가능했지만 지금은 사용할려면 이와 같이 정의를 해야 합니다 마찬가지로 `inputChannel` 은 설정의 `bindings.inputChannel` 와 동일한 값이어야 합니다 

## savingMoney MessageConfig
```
@Configuration
@EnableBinding(CustomSink.class)
public class MessageConfig {


}


```
마찬가지로 bena 을 설정파일에 넣어야 주입이 되고 사용할 수 있습니다 

## savingMoney MiniProjectSavingMoneyApplication

```
@SpringBootApplication
public class MiniProjectSavingMoneyApplication {

	private final static Logger log = LoggerFactory.getLogger(MiniProjectSavingMoneyApplication.class);

	public static void main(String[] args) {
		SpringApplication.run(MiniProjectSavingMoneyApplication.class, args);
	}

	@StreamListener("inputChannel")
	public void loggerSink(EmpChangeModel empChangeModel){

		log.info("Received Message To EmpClient : {} , {}"  , empChangeModel.getStatus() , empChangeModel.getEmd_id());

	}

}


```
소비 입장에서는 특별히 다른건 없이 `@StreamListener("inputChannel")` 를 통해서 카프카가 보내주는 메세지를 받을 준비만 하고 있으면됩니다 그럼 이제 empClient 가 변할때마다 
메세지가 savingMoney 쪽으로 들어오게 되는데 

## 로그
```
c.e.m.MiniProjectSavingMoneyApplication  : Received Message To EmpClient : CREATE , b41fe8fd-fd75-402e-aa60-7b24f276671f
                                           Received Message To EmpClient : READ , b41fe8fd-fd75-402e-aa60-7b24f276671f
                                           Received Message To EmpClient : UPDATE , b41fe8fd-fd75-402e-aa60-7b24f276671f
                                           Received Message To EmpClient : READ , b41fe8fd-fd75-402e-aa60-7b24f276671f

```
이렇게 총 4개의 상태 변화에 따라서 메세지를 소비하고 있습니다 그럼 이것으로 어떻게 어플리케이션간 강한결합을 약하게 만드는지에 대해서는 다음시간에 해보도록 하겠습니다 