---
title: Spring Batch1
author: kimdongy1000
date: 2023-04-22 10:45
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## Batch Processing
일련의 작업이나 프로세스를 한번에 모아서 처리하는 방식을 말합니다 일괄 처리 방식은 주로 대량의 데이터를 처리하거나 반복적인 작업을 자동화 하는데 이를 Batch Processing 라고 합니다 

## 예시 
1. 주문 -> 출고 
    예를들어서 어떤 물류사이트 주문건에 대해서 익일 출고건을 산정할려고 할때 1주문 할때마다 1출고를 올리는 방식이 아닌 오늘 집계된 주문전체건을 통틀어서 익일 출고건을 산정하게 됩니다 
    이때 대량의 데이터를 일련의 방식으로 정의하기 때문에 Batch 를 사용하게 됩니다 

2. 은행 일일 결산 
    은행은 매일 자정부터 짧게는 5분 길게는 30분까지 뱅킹시스템을 막아둡니다 이때 정산Batch를 작업을 하게 되는데 금일 다른 은행으로 부터 이체내역을 산정을 해서 총 은행에서 얼만큼 송금을 보내고 얼만큼 송금을 받는지 집계를 하게 됩니다 이때 일련의 작업을 모든 은행이 동시에 진행을 하기에 Batch 시스템을 가동시켜서 진행을 하게 됩니다 


## Spring Batch
Spring Batch는 기업 시스템의 일상적인 운영에 필수적인 강력한 배치 애플리케이션 개발을 가능하게 하도록 설계된 가볍고 포괄적인 배치 프레임워크입니다. Spring Batch는 사람들이 기대하는 Spring Framework의 특성(생산성, POJO 기반 개발 접근 방식, 일반적인 사용 용이성)을 기반으로 구축되는 동시에 개발자가 필요할 때 더 발전된 엔터프라이즈 서비스에 쉽게 액세스하고 사용할 수 있도록 해줍니다. 

스프링 Batch 의 공식문서에 따르면 <https://docs.spring.io/spring-batch/reference/spring-batch-intro.html> Spring Batch 를 왜 써야 하는지를 적어두었습니다 
앞으로 이 Batch 포스터는 공식문서 +  책 + 다른 사람 블로그 그리고 제 생각을 바탕으로 적어 나갈 예정입니다 

## 작업 환경
```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.batch</groupId>
        <artifactId>spring-batch-test</artifactId>
        <scope>test</scope>      
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    </dependency>
</dependencies>

```
Spring Boot 2.7.1 , JDK 11 의존성은 위와 같이 H2 데이터 베이스 또는 Mysql 를 사용할것입니다 일단 H2 연결을 보고 다음장에서 MySql 연동도 알아보도록 하겠습니다 
Web 을 추가한 이유는 H2 웹 콘솔을 보기 위함입니다 

## Application.yml

```
spring:
    batch:
        jdbc:
            initialize-schema : ALWAYS
    datasource:
        url: jdbc:h2:mem:mydb
        username: sa
        password: password
        driverClassName: org.h2.Driver
    jpa:
        show-sql: true
    h2:
        console:
            enabled : true

```
그리고 Spring Batch 가 DB를 H2 로 사용할 수 있게끔 선언을 해줍니다 그리고 H2는 어플리케이션이 종료되면 메모리 데이터 저장이기 때문에 종료와 함께 데이터는 사라집니다 
그래서 차후 RDBMS 연동도 다루어 볼 것입니다 

## H2 Console 접속

<http://localhost:8080/h2-console/> 로 접속을 하게 되면 다음과 같은 창이 나오게 됩니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ea47f1d5-9ce7-4bc6-9076-4337e40e59a3)

그러면 이런 화면이 보일것인데 이곳에서 우리가 application.yml 에서 적었던 내용을 적어주겠습니다 그리고 접속을 누르게 되면 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/2a58d9c5-99ae-4d05-b47d-588aaf591956)

이런 화면이 나오면 H2 접속까지 된것입니다 그러면 우리는 간단한 job 을 한개 만들어보겠습니다 

## @EnableBatchProcessing

```
@SpringBootApplication
@EnableBatchProcessing
public class SpringBootBatchReviewApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootBatchReviewApplication.class, args);
	}

}

```
이 애너테이션을 달게 되면 Spring 은 다음과 같은 작업을 진행을 하게 됩니다 

1. Spring Batch 의 필수Bean 을 자동으로 구성

2. Spring Batch 의 기본적인 설정을 자동으로 해서 사용자가 큰 설정없이 바로 코드만 집중해서 배치 어플리케션을 만들 수 있게 합니다 

## Hello Spring Batch 작성 

```
package com.example.demo.batch.hello;

@Configuration
@RequiredArgsConstructor
public class HelloSpringBatch {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job helloSpringBatchJob(Step helloStep){

        return jobBuilderFactory.get("helloSpringBatchJob")
                .incrementer(new RunIdIncrementer())
                .start(helloStep)
                .build();
    }

    @Bean
    @JobScope
    public Step HelloStep(){

        return stepBuilderFactory.get("HelloStep")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("Hello Spring Batch Job");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
}


```
이렇게 작성을 하고 실행을 하게 되면 중간에 Hello Spring Batch Job 출력이 되고 종료가 되는것을 알 수 있습니다 

```
2024-02-12 11:44:18.763  INFO 7900 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=helloSpringBatchJob]] launched with the following parameters: [{run.id=1}]
2024-02-12 11:44:18.801  INFO 7900 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [HelloStep]
Hello Spring Batch Job

```

그리고 h2-console 로 가보자 

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/877ea51c-1388-4d18-8bdd-b636810f78bc)

이렇게 우리가 만든적이 없는 테이블 6개가 생겨나게 됩니다 그중 하나를 Selct 했을때 데이터도 들어가 있는 상황을 볼 수 있습니다 설명은 다음 장에서 할 예정이니 이 포스트는 
간단하게 배치프로젝트 만들어서 연동하는것을 진행을 했습니다