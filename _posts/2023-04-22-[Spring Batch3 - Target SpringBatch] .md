---
title: Spring Batch3 - Target SpringBatch
author: kimdongy1000
date: 2023-04-22 13:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## 특정 Batch Job 실행
지난 시간 우리는 간단한 배치 어플리케이션과 배치를 동작시키는 내용에 대해서 살펴보았습니다 오늘은 여러 개의 배치 프로젝트가 있는데 그중에서 특정 배치 프로젝트를 실행시키는 방법에 대해서 알아보도록 하겠습니다

## MultipleSpringBatch 작성 
```
package com.example.demo.batch.multiple;

@Configuration
@RequiredArgsConstructor
public class MultipleSpringBatch {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job multipleBatchJob(Step multipleBatchStep){

        return jobBuilderFactory.get("multipleBatchJob")
                .incrementer(new RunIdIncrementer())
                .start(multipleBatchStep)
                .build();
    }

    @Bean
    @JobScope
    public Step multipleBatchStep(){
        return stepBuilderFactory.get("multipleBatchStep")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("multipleBatchStep Start");
                        return null;
                    }
                }).build();
    }
}


```
지난 시간에 같은 파일 위치로 해서 패키지 위치만 달리해서 만들어보자 현재 이 프로젝트에는 지난 시간에 만들었던 HelloSpringBatch 도같이 있다 이렇게 하고 작성을 하면 Batch는 아래 모든 배치를 동작시키게 됩니다

```
2024-02-14 22:40:13.891  INFO 11052 --- [           main] o.s.b.a.b.JobLauncherApplicationRunner   : Running default command line with: []
2024-02-14 22:40:14.046  INFO 11052 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=helloSpringBatchJob]] launched with the following parameters: [{run.id=1}]
2024-02-14 22:40:14.088  INFO 11052 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [HelloStep]
Hello Spring Batch Job
2024-02-14 22:40:14.119  INFO 11052 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [HelloStep] executed in 31ms
2024-02-14 22:40:14.124  INFO 11052 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=helloSpringBatchJob]] completed with the following parameters: [{run.id=1}] and the following status: [COMPLETED] in 57ms
2024-02-14 22:40:14.131  INFO 11052 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=MultipleBatchJob]] launched with the following parameters: [{run.id=1}]
2024-02-14 22:40:14.141  INFO 11052 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [multipleBatchStep]
multipleBatchStep Start
2024-02-14 22:40:14.146  INFO 11052 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [multipleBatchStep] executed in 4ms
2024-02-14 22:40:14.150  INFO 11052 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=MultipleBatchJob]] completed with the following parameters: [{run.id=1}] and the following status: [COMPLETED] in 17ms

```

물론 이렇게 동작을 시켜도 되지만 우리는 특정 하나의 batch만 동작을 시키기 위해 아래처럼 수정을 할 것입니다

## job.name 추가 
```

spring:
    batch:
        job:
            names : ${job.name:NONE}
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
application.yml로 와서 상단에 spring.batch.job.name 을 추가해 주자 이제 이러면 batch 프로젝트는 동적으로 진행할 batch 프로젝트를 파라미터를 통해서 받을 수 있다 그러면 우리는 어떻게 넣는가?
이클립스 기준과 인텔리j 기준 둘 다 설명을 할 것이다

## 인텔리J 기준 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/3f138b2a-546e-4c78-914f-be12e9c2c2c1)

상단에 저 부분을 선택해서 Edit configurtion 창을 연다

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/b4cf8a8b-f30a-404c-b5ff-1137f00dc180)

이렇게 안에 파라미터에 --spring.batch.job.names=helloSpringBatchJob 이렇게 적어주면 된다 이때 helloSpringBatchJob의 값의 기준은 실행할 job의 이름을 가져오면 된다

`return jobBuilderFactory.get("helloSpringBatchJob")` , `return jobBuilderFactory.get("multipleBatchJob")` 말이다 

## STS 이클립스 기준

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/efc0472e-8d1b-4c4c-88ff-5dc9232dbd2e)

이클립스나 STS는 하단 RunConfiguration 클릭

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/4c075304-dd46-4db9-ae0b-b0df0c5e0e0d)

그리고 이 창을 열어서 이렇게 vmOption에 -Dspring.batch.job.names=multipleBatchJob 게 적어주면 된다

이는 인텔리j 하고 창하고 넣는 방식과 값이 다르기에 주의가 필요하다

오늘은 하나의 프로젝트에 여러 개의 batch에 있을 때 타깃을 정해서 실행시키는 방법에 대해서 알아보았습니다

