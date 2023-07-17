---
title: Spring Batch1
author: kimdongy1000
date: 2023-04-22 10:45
categories: [Back-end, Spring - MVC]
tags: [ MVC , Bactch ]
math: true
mermaid: true
---

## Batch
Spring Batch 는 대용량 데이터 처리와 일괄작업에 특화된 Spring 프레임워크중 하나입니다 

## 왜 사용할까?
웹어플리케이션에 심심찮게 등장하는 batch 왜 사용하고 언제 사용해야 할까? 아마 웹어플리케이션을 개발하다 보면 다음과 같은 문제에 마주할 수 있다 
- 수백만건의 데이터를 insert 
- 수백만건의 데이터를 핸들링 
- 주기적인 작업 
- insert 데이터 검증 

예를 들어 위와 같은 작업을 batch 에 일임하지 않고 클라이언트가 직접 http 요청을 한다고 하자 예를 들어서 다음처럼 

```

element.addEventListener('click' , (e) => {

   try {
        const response = await fetch("https://example.com/profile", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
        },
            body: JSON.stringify(data),
        });

        const result = await response.json();
        console.log("성공:", result);
    } catch (error) {
        console.error("실패:", error);
    }
})

```

예를 들어서 현재 이 api 가 대량의 데이터를 insert 하는 작업이라고 생각하자 이떄 이 insert 데이터는 이미 DB 에 insert 되어 있는 데이터를 가공해서 새로운 테이블에 넣어준다는 시나리오이다 그런 api 를 직접 개발해서 http 요청을 준다고 생각을 하자 현재 이 fetch 는 기본 async 가 true 이기 때문에 이 작업을 진행하는 동안 해당 페이지에서 다른 작업을 진행할 수 있지만 만약 페이지를 벗어나면 요청한 작업은 계속 진행중이지만 끝나는 요청결과를 받지 못하게 된다 그렇다고 수백만건이 언제 끝날지도 모르는 시간에 계속 이 페이지에 상주할 수 없는 노릇이다 이때 필요한것이 batch 이다 

그럼 우리가 batch 시나리오를 어떻게 개발을 해야 할지 감이 잡혔으면 batch 에 대한 몇가지 용어를 알아보자 

## batch 용어 

1. Job 
    Spring Batch 의 기본개념중 하나로 하나 이상의 Step 로 구성된 배치작업을 의미합니다 Job 은 실행가능한 단위입니다 

2. Step 
    Job 을 구성하는 단위 작업단위 입니다 이때 이 작업단위를 좀더 쪼개서 보면 
    1. ItemReader
        Step 의 첫번째 구성요소로 데이터를 읽어오는 역활을 합니다 주로 데이터소스에서 데이터를 읽어오는 역활을 수행합니다 
    2. ItemProcessor
        ItemReader 에서 읽어온 데이터를 가공하거나 변환하는 역활을 합니다 데이터의 유효성 검사 , 필터링 변환 등의 작업을 수행할 수 있습니다 
    3. ItemWriter  
        ItemProcessor 처리된 데이터를 저장하거나 외부시스템에 전달하는 역활 
    4. Chunk 
        Step 의 처리 단위를 나타냅니다 일정량의 데이터를 한번에 처리하는 단위입니다 예를 들어서 100개의 단위를 Chunk 단위로 처리하면 ItemReader는 100개의 데이터를 읽어오고 ItemProcessor ItemReader 가 읽어온 데이터를 처리하고 ItemWriter 가공합니다 Chunk 는 유동적으로 조절할 수 있지만 한번에 처리할 개수가 많으면 많을 수록 성능에 영향을 미치기에 철저한 분석이 필요합니다 
    5. StepListener
        Step 의 실행 중 발생하는 이벤트에 대한 리스너입니다 Step 의 시작 , 종료 , 익셉션 발생 등의 이벤트를 처리하고 원하는 로직을 수행할 수 있습니다 

3. JobRepository 
    Spring Batch 가 Job 실행의 메타데이터를 관리하는데 사용됩니다 Job의 실행 상태, Step의 상태, 실행 완료된 Job 인스턴스 등의 정보를 저장하고 추적합니다.

4. ExecutionContext 
    Job 실행 중에 Step 간에 데이터를 공유하기 위해 사용되는 컨텍스트입니다. ExecutionContext는 Job 또는 Step에 대한 상태 정보를 포함하고 있으며, 데이터를 저장하고 검색할 수 있습니다.
5. JobLauncher 
    Job을 실행하는 역할을 합니다. JobLauncher는 Job을 실행하고 실행 결과를 반환합니다.    

## batch 프로젝트 시작

가장 간단한 예시로 시작을 하자 뭐든 가장 간단한 프로젝트를 만든뒤 이를 계속 발전해 나가는게 개발자의 공부 및 덕목이다 

Maven
```

<dependencies>

    <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-batch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>


</dependencies>

```

의존성은 기본적으로 spring-boot-starter-batch 만 있으면 되지만 batch 특정상 DB 와 관련있는 작업이 많다보니 자동으로 jdbc 라이브러리를 의존성으로 들어오게 되는데 
당장 구축해놓은 DB 가 없다면 우리는 자동 DB 설정을 하지 못하게 다음과 같은 명령어를 달고 가면된다 

`@SpringBootApplication(exclude={DataSourceAutoConfiguration.class})`

그리고 간단한 예제를 보면 

```

package com.cybb.main.config;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.support.ListItemReader;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


import java.util.Arrays;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public ItemReader<String> itemReader() {
        return new ListItemReader<String>(Arrays.asList("Data 1", "Data 2", "Data 3"));
    }

    @Bean
    public ItemProcessor<String, String> itemProcessor() {
        return item -> item.toUpperCase();
    }

    @Bean
    public ItemWriter<String> itemWriter() {
        return items -> {
            for (String item : items) {
                System.out.println("Writing item: " + item);
            }
        };
    }

    @Bean
    public Step step(ItemReader<String> itemReader, ItemProcessor<String, String> itemProcessor, ItemWriter<String> itemWriter) {
        return stepBuilderFactory.get("step")
                .<String, String>chunk(1)
                .reader(itemReader)
                .processor(itemProcessor)
                .writer(itemWriter)
                .build();
    }

    @Bean
    public Job job(Step step) {
        return jobBuilderFactory.get("job")
                .start(step)
                .build();
    }



}


```

자 그럼 설명을 하기 전에 결과를 먼저보자 

```

2023-05-30 11:09:04.174  INFO 11132 --- [           main] o.s.b.a.b.JobLauncherApplicationRunner   : Running default command line with: []
2023-05-30 11:09:04.176  WARN 11132 --- [           main] o.s.b.c.c.a.DefaultBatchConfigurer       : No datasource was provided...using a Map based JobRepository
2023-05-30 11:09:04.177  WARN 11132 --- [           main] o.s.b.c.c.a.DefaultBatchConfigurer       : No transaction manager was provided, using a ResourcelessTransactionManager
2023-05-30 11:09:04.215  INFO 11132 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : No TaskExecutor has been set, defaulting to synchronous executor.
2023-05-30 11:09:04.303  INFO 11132 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] launched with the following parameters: [{}]
2023-05-30 11:09:04.371  INFO 11132 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step]
Writing item: DATA 1
Writing item: DATA 2
Writing item: DATA 3
2023-05-30 11:09:04.444  INFO 11132 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step] executed in 72ms
2023-05-30 11:09:04.458  INFO 11132 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 100ms

```
뭐가 먼지는 모르겠지만 일단 눈에 띄는건 

```
Writing item: DATA 1
Writing item: DATA 2
Writing item: DATA 3

```
이렇게 DATA1 , DATA2 , DATA3 이렇게 찍고 끝난것이 보인다 그럼 우리가 앞에서 설명과 연계해서 소스를 직접 분석을 해보자 

```

@Configuration
@EnableBatchProcessing
public class BatchConfig {}

```

먼저 클래스 단위로 살펴보면 이 아래 메서드는 전부 @Bean 으로 정의되어 있기 때문에 spring 에게 Bean 을 읽게끔 해야 한다 그래서 이 파일이 설정파일임을 알리는 
@Configuration 필수이고 @EnableBatchProcessing 애노테이션을 통해서 이 클래스파일은 BatchJob 이라는 것을 알려주는것이다 

## ItemReader

```

@Bean
public ItemReader<String> itemReader() {
    return new ListItemReader<String>(Arrays.asList("Data 1", "Data 2", "Data 3"));
}

```

ItemReader Step 의 첫번째 구성요소로 데이터를 읽어오는 역활을 합니다 주로 데이터소스에서 데이터를 읽어오는 역활을 수행합니다 
우리는 당장구축을 해놓은 데이터가 없기 때문에 당장은 임의로 만든 데이터를 ItemReader 삽입을 해서 사용을 하겠습니다 


## ItemProcessor
```

@Bean
public ItemProcessor<String, String> itemProcessor() {
    return item -> item.toUpperCase();
}

```
ItemReader 에서 읽어온 데이터를 가공하거나 변환하는 역활을 합니다 데이터의 유효성 검사 , 필터링 변환 등의 작업을 수행할 수 있습니다  
우리는 이 변환을 통해서 대소문자 데이터 (ItemReader Arrays.asList("Data 1", "Data 2", "Data 3")) 를 완전 대문자로 변환하는 작업입니다 


## ItemWriter

```

@Bean
public ItemWriter<String> itemWriter() {
    return items -> {
        for (String item : items) {
            System.out.println("Writing item: " + item);
        }
    };
}

```

ItemWriter 는 ItemProcessor 처리된 데이터를 저장하거나 외부시스템에 전달하는 역활 우리는 당장 저장하거나 , 전달이 없으니 다음처럼 콘솔에 출력을 하는것으로 



## Step
```

@Bean
public Step step(ItemReader<String> itemReader, ItemProcessor<String, String> itemProcessor, ItemWriter<String> itemWriter) {
    return stepBuilderFactory.get("step")
            .<String, String>chunk(1)
            .reader(itemReader)
            .processor(itemProcessor)
            .writer(itemWriter)
            .build();
}

```
Job 을 구성하는 단위 작업단위 입니다 이때 각 단위마다 해야할일을 정의해두고 있습니다 job 은 step 를 가져올것이고 이때 한번에 처리할 단위는 1(Chunk)
그리고 데이터를 어디서 읽어오는지는 reader , 그리고 읽어온 데이터 전처리는 processor 그리고 전처리 데이터 후처리는 writer 로 설정하는 빌더패턴으로 설정 
그렇게 빌더패턴으로 정의한 뒤 


## Job
@Bean
public Job job(Step step) {
    return jobBuilderFactory.get("job")
            .start(step)
            .build();
}

Spring Batch 의 기본개념중 하나로 하나 이상의 Step 로 구성된 배치작업을 의미합니다 Job 은 실행가능한 단위입니다 
job을 실행하게 됩니다 이렇게 보면 배치는 정말 간단해 보이지만 사실은 엄청 복잡하고 대용량 데이터에 적합한 프레임워크 단위라서 현재 단위는 별로 알맞지 않다 
다음 장에서는 엄청단 대량 데이터를 발생시켜서 그것을 직접 전처리 하는 작업을 진행할것이다 




















