---
title: Spring Batch2 - Hello SpringBatch2
author: kimdongy1000
date: 2023-04-22 12:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

우리는 지난 시간에 프로젝트 생성과 간단한 배치 프로그램을 만들어서 실행을 해보았다 이 포스트는 그에 대한 설명을 조금 하고 다음 장으로 넘어갈 예정입니다

## Batch 아키텍쳐

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0f23110f-4499-472a-9d95-af63762ff5c7)

이것이 spring Batch 이 아키텍쳐입니다 출처 : <https://docs.spring.io/spring-batch/reference/domain.html>

## JobRepository
Spring Batch 가 실행될 때마다 배치작업 상태를 추적하고 관리하기 위한 인터페이스입니다

1. 작업 실행상태 관리
    작업이 실행될 때 JobRepository는 해당 작업의 실행 상태를 추적합니다. 이때 Job의 모든 상태를 저장을 하게 됩니다 (실행 시간, 종료시간, 성공, 실패 등..)

2. 트랜잭션 관리
JobRepository는 작업 실행 중 발생하는 모든 변경 사항을 트랜잭션으로 관리하여 데이터 일관성을 유지합니다.

3. 작업 재시도
작업이 실패한 경우 JobRepository를 사용하여 작업을 재시도할 수 있습니다. 작업이 실패하면 JobRepository는 해당 작업의 상태를 실패로 표시하고, 필요한 경우 작업을 다시 시작하거나 재시도할 수 있도록 지원을 합니다

## Job
Batch 작업의 최상위 단위로 전체 배치 프로세서를 캡슐화한 엔티티입니다 해당 Job 은 이름과 하나 이상의 Step으로 구성이 되어 해당 Job 이 무슨 역할을 하는지 명확하게 알 수 있습니다

```
@Bean
public Job helloSpringBatchJob(Step helloStep){

    return jobBuilderFactory.get("helloSpringBatchJob")
            .incrementer(new RunIdIncrementer())
            .start(helloStep)
            .build();
}

```
위 코드를 보면 `jobBuilderFactory.get("helloSpringBatchJob")` 이 job의 이름이 됩니다 그리고 start 안에 helloStep를 넣어서 해당 Job의 시작을 helloStep로 시작을 하겠다는 뜻입니다 


## Step 
배치 작업의 독립적인 순차적 단계를 캡슐화한 도메인 객체입니다 이 Step 안에는 실제 배치 작업을 어떻게 정의하고 제어하는지 필요한 소스코드를 입력하게 됩니다 이 Step이라는 곳에서
모든 배치 일괄작업 특히나 비즈니스 작업이 주로 이루어지게 됩니다 역시나 이름과 해야 할 일을 정의를 해두게 됩니다 이곳에서는 Tasklet으로 해당 step 이 무엇을 해야 하는지 정의를 해두었습니다

```

@Bean
@JobScope
public Step helloStep(){

    return stepBuilderFactory.get("helloStep")
        .tasklet(new Tasklet() {
            @Override
            public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                System.out.println("Hello Spring Batch Job");
                return RepeatStatus.FINISHED;
            }
        }).build();
}

```

그럼 이 소스의 전체적인 흐름을 보게 되면

Job 정의
이름 : helloSpringBatchJob
시작 : helloStep

Step 정의
이름 : HelloStep
해야 할 일 : 콘솔에 "Hello Spring Batch Job"를 출력하고 종료

실행 시 Job 트리거가 Job 을 실행을 하고 job 은 안에 step의 순서대로 실행을 하게 됩니다

## 스키마 설명 (Sprinb Batch 메타데이터)

1. BATCH_JOB_EXECUTION : job 이 실행이 될 때마다 해당 작업의 실행 정보를 저장합니다

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/509dade5-9c3f-4532-9ffd-1a63e9eab9d9)

이곳을 보게 되면 언제 생성이 되었고 (CREATE_TIME) 언제 실행이 되었고 (START_TIME) 언제 끝났는고 (END_TIME) 끝났을 때의 상태를 (STATUS , EXIT_CODE) 각각 저장을 해둡니다
그 외에도 JOB_EXECUTION_ID는 배치 작업의 고유 식별자입니다 실행을 할 때마다 자동으로 증가하는 일련번호가 할당이 됩니다

2. BATCH_JOB_EXECUTION_CONTEXT :각 배치 작업에 대한 실행 컨텍스트를 저장하는데 활용합니다

3. BATCH_JOB_EXECUTION_PARAMS : 배치작업 실행에 필요한 파라미터를 저장하여 차후 재시도할 때 해당 파라미터를 그대로 사용하여 작업의 재실행이나 상태 추적 등에 사용 등에 사용됩니다

4. BATCH_JOB_INSTANCE : 각 배치 작업의 실행에 대한 인스턴스 정보를 저장합니다.

5. BATCH_STEP_EXECUTION : 배치에서 Step의 실행 이력을 추적하기 위해 사용되는 테이블입니다

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/54c0192d-60d1-4f80-a66c-4b7cf260764f)
이곳을 보게 되면 Step 언제 시작이 되었고 (START_TIME) 언제 끝났고 (END_TIME) 스텝의 이름이 무엇인지 (STEP_NAME) 그리고 Step 이 끝났을 때의 상태가 무엇인지 (STATUS) 등 데이터를 저장을 해둡니다

6. SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT : Step 실행 중에 필요한 임시 데이터나 상태 정보를 저장하여 나중에 작업의 재실행이나 추적 등에 사용이 됩니다

이 포스트에서는 이전 포스트에서 다루었던 Batch에 대한 간단한 설명과 스키마의 설명에 대해서 이어나갔습니다 앞으로 나오는 Batch와 관련된 정보는 그때그때마다 설명을 하고
다음 포스트에는 여러 가지 Job 들 중에서 하나를 실행시킬 때 필요한 커맨드 라인 배치 실행에 대해서 알아보겠습니다