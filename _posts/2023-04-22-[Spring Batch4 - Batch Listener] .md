---
title: Spring Batch4 - Batch Listener
author: kimdongy1000
date: 2023-04-22 14:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## Batch Listener
이번 시간에는 Batch Listener에 대해서 알아보겠습니다 잠깐 Listener에서 소개를 하자면 영어 뜻으로는 경청자를 뜻을 가지고 있지만 프로그래밍에서는 어떤 이벤트가 발생하는 것을 귀 기울여 듣다가 특정 이벤트 발생 시 실행되는 것을 뜻합니다 이를 Spring Batch 와 결합을 해서 설명을 하면 다양한 Listener 이 있겠지만 Batch에서만큼은 성공과 실패에 대한 Lisnter 을 작성해 보도록 하겠습니다

```
@Configuration
@RequiredArgsConstructor
public class ListenerBatch {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job listenerBatchJob(Step listenerBachStep){

        return jobBuilderFactory.get("listenerBatchJob")
                .incrementer(new RunIdIncrementer())
                .start(listenerBachStep)
                .listener(new JobListener())
                .build();
    }

    @Bean
    @JobScope
    public Step listenerBachStep(){

        return stepBuilderFactory.get("listenerBachStep")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {

                        for(int i = 0; i < 10; i++){
                            System.out.println("Time is " + i);
                        }
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
}



```
우리는 그냥 평범한 job 을 하나 만들 것이다 TaskLet에선 1부터 10까지 값을 출력하는 작업을 만들 것이고 여기서 Listener 을 보게 되면 `.listener(new ListenerBatch())` job에 Listener 을 등록하게 된다 그럼 이 구현체를 한번 보자


## JobExecutionListener
```
@Slf4j
public class JobListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("Job started: " + jobExecution.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {

        System.out.println("Job completed: " + jobExecution.getJobInstance().getJobName());
        
        if (jobExecution.getStatus().isUnsuccessful()) {
            System.out.println("Job failed with status: " + jobExecution.getStatus());
        } else {
            System.out.println("Job finished successfully");    
        }

    }
}

```

이는 job에 등록할 Listener이다 JobExecutionListener를 상속받아서 구현을 해야 하는 구현체이다 JobExecutionListener 은 Job 실행하기 전 메서드와 실행이 끝난 후 메서드 `beforeJob` , `afterJob` 을 제공하고 있습니다 지금 이 코드는 간단하게 Job 을 실행하기 전에 어떤 Job 이 실행이 되는지 적어 주었고 그리고 끝이 나게 되면 어떤 Job 이 끝났는지와 성공 실패 여부에 따라서 다른 문구를 출력하게끔 만들었습니다 그럼 실행을 하게 되면 

## beforeJob
배치 작업이 실행되기 전에 호출되는 메서드로 배치작업에 필요한 초기화 데이터 등을 선언해서 지행을 할 수 있습니다

## afterJob
배치 작업이 완전히 끝난 후 호출되는 메서드로 이때 배치의 성공유무와 관계없이 호출이 되며 이때는 Job의 실행상태, 결과 등을 확인할 수 있습니다

## 실행결과 

```
2024-02-17 20:18:15.577  INFO 7488 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=listenerBatchJob]] launched with the following parameters: [{run.id=1}]
Job started: listenerBatchJob
2024-02-17 20:18:15.636  INFO 7488 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [listenerBachStep]
Time is 0
Time is 1
Time is 2
Time is 3
Time is 4
Time is 5
Time is 6
Time is 7
Time is 8
Time is 9
2024-02-17 20:18:15.658  INFO 7488 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [listenerBachStep] executed in 22ms
Job completed: listenerBatchJob
Job finished successfully
2024-02-17 20:18:15.667  INFO 7488 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=listenerBatchJob]] completed with the following parameters: [{run.id=1}] and the following status: [COMPLETED] in 70ms
```

이렇게 볼 수 있다 이를 보면 job 을 실행하기 전에 Job started: listenerBatchJob 이 호출되는 것을 보았고 Tasklet 이 호출이 완전히 끝이 나고 Job finished successfully 이 호출되는 것을 보았다

그럼 실패 환경을 만들어보자

## 실패환경
```
for(int i = 0; i < 10; i++){

    if(i == 8) throw new RuntimeException("No more than 8");
    System.out.println("Time is " + i);

}

```

반복문 8이 될 때 에러를 던지게 되면 이는 실패가 될 것이다 그럼 한번 다시 실행을 해보자

```
Job started: listenerBatchJob
2024-02-17 20:21:36.935  INFO 5368 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [listenerBachStep]
Time is 0
Time is 1
Time is 2
Time is 3
Time is 4
Time is 5
Time is 6
Time is 7
2024-02-17 20:21:36.955 ERROR 5368 --- [           main] o.s.batch.core.step.AbstractStep         : Encountered an error executing step listenerBachStep in job listenerBatchJob

EXCEPTION ....

2024-02-17 20:21:36.960  INFO 5368 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [listenerBachStep] executed in 24ms
Job completed: listenerBatchJob
Job failed with status: FAILED

```
Job 을 실행하기 전의 before 메서드는 그대로 동작하지만 에러가 발생이 되면 after에서는 분기 처리에 따라서 실행을 다른 메시지를 보여줄 수 있게 됩니다
여기까지는 간단한 배치 어플리케이션인데 다음 시간에는 이 Listner 을 활용해서 오류가 났을 때 이메일 발송 또는 Slack에 연동해서 같이 있는 사람에게 메시지를 전달하는 방법에 대해서 코딩을 해보도록 하겠습니다

