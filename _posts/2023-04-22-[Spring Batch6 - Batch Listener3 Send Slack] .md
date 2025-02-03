---
title: Spring Batch6 - Batch Listener3 Send Slack
author: kimdongy1000
date: 2023-04-22 16:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## 성공 실패시 Slack 연동하기
앞에서는 이메일을 통해서 batch 의 성공여부를 파악했는데 이번에는 slack 로 메세지를 한번 내보내겠습니다 slack 은 직장에서 프로젝트 단위로 대화방을 만들어서 특정 목적을 공유하는 직원들이 만든 단체 메신저 방입니다 

## slack api 활용하기 
우리는 slack bot 을 이용할것입니다 bot 이 batch 에 대한 결과물을 지속적으로 전달하는 방향으로 개발을 하겠습니다 
<https://api.slack.com/> 에 접속을 합니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/1efaceda-a040-475d-87e4-a70fd639286b)

1. 우측상단에 Your app 클릭 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/bb9c8704-c115-40cc-9ad9-db1c36161404)

2. 중앙 Create an App 클릭

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/fedca3ba-8484-4c7a-8c1c-3b7ef335280c)

3. From scratch 클릭 

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/01aac8eb-9541-43fb-9d55-d091c90e6ada)

4. App 이름 정하기 

저는 간단하게 Spring_Batch_Listener 라고 짓겠습니다 그리고 새 워크 스페이스로 해서 create App 생성 

5. Bot 설정 

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/5743c282-1fab-4d40-82d1-0248daddd7b2)

해당페이지 하단에 있는 Bot 을 클릭합니다 

6. OAtuh & Permissions 설정하기

인증을 OAuth 토큰으로 인증을 받아서 해당 Bot 을 컨트롤할 수 있게 토큰을 발급받을것입니다 물론 이때 발급받는 secret 토큰은 노출이 되면 안됩니다 그렇게 되면 다른 사람이 악의적인 마음으로 시크릿 토큰으로 나의 slack bot 을 조종 할 수 있습니다

7. Scopes 설정

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ba80091c-137b-4d3d-9ed6-9030f6d0a795)

Oauth 에서 해당 시크릿 코드에 권한을 부여해서 특정 권한 이상을 행세하지 못하게 하는데 우리는 두개의 scope 를 설정하겠습니다 
메세지를 전달하는 scope 와 , 기본적인 체널의 정보를 읽어오는 scope 를 추가하겠습니다

8. Secret Token 발급하기 

![7](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/3f2717fc-cfbe-4e56-8867-c6c48df2371f)

위로 올라가면 이제 Secret Token 을 발급받을 수 있는 창이 있습니다 이를 클릭해서 토큰을 발급받습니다 

9. 워크 스페이스 확인

![8](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/88109eb7-b3ea-4ec7-9e58-2c3b3b65e239)

지금 이 모습은 프로그램까지 짜서 만든 완성본이다 그럼 시작을 해보자

10. 채널 만들기 

![9](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/6ace3468-eaa9-4dd7-934b-0a319b637939)

워크스페이스 상단에 채널을 만들고 그 안에서 우리가 만든 Bot 을 채널에 참여시키면 됩니다 




## Slack 메세지 전달 하는 프로그램 작성

SDK 참조사이트 : <https://slack.dev/java-slack-sdk/guides/getting-started-with-bolt#gradle> 

해당 SDK를 참조해서 개발을 진행을 하시면됩니다 위의 버전은 gradle 인데 저는 maven 으로 진행을 하겠습니다 

## POM.xml 의존성 추가


```

<!-- https://mvnrepository.com/artifact/com.slack.api/bolt -->
<dependency>
    <groupId>com.slack.api</groupId>
    <artifactId>bolt</artifactId>
    <version>1.38.2</version>
</dependency>

<!-- https://mvnrepository.com/artifact/com.slack.api/bolt-servlet -->
<dependency>
    <groupId>com.slack.api</groupId>
    <artifactId>bolt-servlet</artifactId>
    <version>1.38.2</version>
</dependency>

<dependency>
    <groupId>com.slack.api</groupId>
    <artifactId>bolt-jetty</artifactId>
    <version>1.38.2</version>
</dependency>



```

slack 에 대한 의존성은 이정도만 추가를 하면된다 다른 의존성들은 앞에서 사용했던 batch 에 관련한 의존성임으로 생략하겠습니다

## Slack_Message_Service
```
@Component
public class Slack_Message_Service {

    @Value("${slack.token}")
    private String token;

    public void sendMessageToSlack(String message){

        String slack_chanel_name = "#배치모니터링";

        try{

            MethodsClient methods = Slack.getInstance().methods(token);

            ChatPostMessageRequest request = ChatPostMessageRequest.builder()
                    .channel(slack_chanel_name)
                    .text(message)
                    .build();

            methods.chatPostMessage(request);



        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}


```

이곳에서 slack 에 메세지를 보낼것입니다 이때는 token 은 application.yml 에 저장을 하고 사용할 예정입니다 이 서비시를 호출하면 해당 slack 채널에 메세지가 나가게 됩니다 

## Listener_SlackBatch
```
@Configuration
@RequiredArgsConstructor
public class Listener_SlackBatch {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private Slack_Message_Service slackMessageService;

    @Bean
    public Job listener_slackBatch(Step listener_slackStep){

        return jobBuilderFactory.get("listener_slackBatch")
                .incrementer(new RunIdIncrementer())
                .start(listener_slackStep)
                .listener(new Listener_slackListener(slackMessageService))
                .build();
    }

    @Bean
    public Step listener_slackStep(){

        return stepBuilderFactory.get("listener_slackStep")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                        for(int i = 0; i < 10; i++){
                            System.out.println("i : count: " +   i);
                        }
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
}


```

배치 프로그램을 만듭니다 이 프로그램은 slack 과 메세지 연동이 주 이기 떄문에 step 하고 이런것들은 간단하게 작성을 하겠습니다 

## Listener_slackListene
```
public class Listener_slackListener implements JobExecutionListener {

    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH시 mm분 ss초");
    private Slack_Message_Service slackMessageService;

    public Listener_slackListener(Slack_Message_Service slackMessageService){
        this.slackMessageService = slackMessageService;
    }



    @Override
    public void beforeJob(JobExecution jobExecution) {

        String text = LocalDate.now() + " " + LocalTime.now().format(formatter) + " " +jobExecution.getJobInstance().getJobName() + " 실행되었습니다";
        slackMessageService.sendMessageToSlack(text);

    }

    @Override
    public void afterJob(JobExecution jobExecution) {

        String text = LocalDate.now() + " " + LocalTime.now().format(formatter) + " " +jobExecution.getJobInstance().getJobName();

        if (jobExecution.getStatus().isUnsuccessful()) {
            text = text+ "job 실행이 실패 되었습니다";
        } else {
            text = text+ "job 실행이 성공 되었습니다";

        }

        slackMessageService.sendMessageToSlack(text);


    }
}


```

그리고 제일 중요한 batch_listner 을 작성을 해줍니다 그리고 listner 이 호출될때마다 작성한 메세지를 send 를 통해서 slack 에 전달을 해줍니다 그러면 


![8](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/88109eb7-b3ea-4ec7-9e58-2c3b3b65e239)

채널에 참여한 인원은 우리가 만든 봇에 의해서 전체적으로 메세지를 받을 수 있습니다


