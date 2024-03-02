---
title: Spring Batch5 - Batch Listener Send Email
author: kimdongy1000
date: 2023-04-22 15:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## 성공 실패시 이메일 연동하기 
거창하게 메일 서버를 만들어서 사용하기엔 무리가 있으므로 G메일의 SMTP 를 활용을 하겠습니다 

## maven 추가
```
<dependency>
    <groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-mail</artifactId>
	
```
spring boot 기준으로 mail starter 을 받아줍니다 그리고 gmail smtp 설정을 진행을 합니다 이때 구글에서 gamil 사용하겠다고 설정하는것은 검색하면 나옴으로 그 부분은 생략하겠습니다 (앱 2차 비밀번호 사용 등)

## application.yml 설정
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
    mail:
        protocol : smtp
        host: smtp.gmail.com
        port: 587
        username: [Gmail 아이디]@gmail.com
        password: [발급받은 2차 비밀번호 또는 gmail 비밀번호]
        default-encoding: utf-8
        properties:
            mail:
                smtp:
                    auth: true
                    starttls:
                        enable: true
```
yml 을 수정할것입니다 이때 값들을 이와 같이 설정하고 username , password 는 본인의 username , 발급받은 password 를 사용하면됩니다 


## MailConfig 작성 

```
@Configuration
public class MailConfig {

    @Value("${spring.mail.protocol}")
    private String protocol;

    @Value("${spring.mail.host}")
    private String host;

    @Value("${spring.mail.port}")
    private int port;

    @Value("${spring.mail.username}")
    private String username;

    @Value("${spring.mail.password}")
    private String password;

    @Value("${spring.mail.default-encoding}")
    private String default_encoding;

    @Value("${spring.mail.properties.mail.smtp.auth}")
    private String auth;

    @Value("${spring.mail.properties.mail.smtp.starttls.enable}")
    private String enable;


    @Bean
    public JavaMailSenderImpl javaMailSenderImpl(Properties mailProperties){

        JavaMailSenderImpl javaMailSender = new JavaMailSenderImpl();


        javaMailSender.setHost(host);
        javaMailSender.setPort(port);
        javaMailSender.setUsername(username);
        javaMailSender.setPassword(password);

        javaMailSender.setJavaMailProperties(mailProperties);

        return javaMailSender;
    }

    @Bean
    public Properties mailProperties (){

        Properties properties = new Properties();
        properties.setProperty("mail.smtp.auth" , auth);
        properties.setProperty("mail.smtp.starttls.enable" , enable);

        return properties;
    }
}


```
JavaMailSenderImpl 을 bean 으로 만들어서 공통으로 사용해서 호출할것입니다 그래서 config 파일 작성을 해서 진행을 하겠습니다 


## Job 만들기 
```
@Configuration
@RequiredArgsConstructor
public class Listener_emailBatch {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private JavaMailSenderImpl javaMailSender;


    @Bean
    public Job listener_emailJob(Step listener_emailStep){

        return jobBuilderFactory.get("listener_emailJob")
                .incrementer(new RunIdIncrementer())
                .start(listener_emailStep)
                .listener(new Listener_emailListenerBatch(javaMailSender))
                .build();
    }

    @Bean
    @JobScope
    public Step listener_emailStep(){

        return stepBuilderFactory.get("listener_emailStep")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                        for(int i = 0; i < 10; i++){

                            //if(i == 8) throw new RuntimeException("No more than 8");
                            System.out.println("Time is Email Count " + i);

                        }
                        return RepeatStatus.FINISHED;

                    }
                }).build();
    }
}


```
job 은 지난시간과 동일한 step 을 가지는 job 을 만들고 이때 사용하는 listener 도 만들겠습니다 이때 중요하것은 이곳에서 JavaMailSenderImpl 을 주입받아서 listener 생성자로 초기화 할 예정입니다 

## BatchListener

```
public class Listener_emailListenerBatch implements JobExecutionListener {

    private JavaMailSenderImpl javaMailSender;

    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH시 mm분 ss초");


    public Listener_emailListenerBatch(){

    }

    public Listener_emailListenerBatch(JavaMailSenderImpl javaMailSender){

        this.javaMailSender = javaMailSender;
    }



    @Override
    public void beforeJob(JobExecution jobExecution) {

        try{
            MimeMessage message = javaMailSender.createMimeMessage();
            MimeMessageHelper messageHelper = new MimeMessageHelper(message, false, "UTF-8");

            String subject = LocalDate.now() + " " + LocalTime.now().format(formatter) + " " +jobExecution.getJobInstance().getJobName() + "시작 알림 메일";
            String text = LocalDate.now() + " " + LocalTime.now().format(formatter) + " " +jobExecution.getJobInstance().getJobName() + " 실행되었습니다";

            messageHelper.setTo(""); // 전송할 이메일 주소 (받는 사람)
            messageHelper.setSubject(subject);
            messageHelper.setText(text);

            javaMailSender.send(message);

        }catch(Exception e){
            e.printStackTrace();
        }

    }

    @Override
    public void afterJob(JobExecution jobExecution) {


        try{
            MimeMessage message = javaMailSender.createMimeMessage();
            MimeMessageHelper messageHelper = new MimeMessageHelper(message, false, "UTF-8");
            String subject = LocalDate.now() + " " + LocalTime.now().format(formatter) + " " +jobExecution.getJobInstance().getJobName();
            String text = LocalDate.now() + " " + LocalTime.now().format(formatter) + " " +jobExecution.getJobInstance().getJobName();

            if (jobExecution.getStatus().isUnsuccessful()) {

                subject = subject + "실행 실패 알림";
                text = text+ "job 실행이 실패 되었습니다";

            } else {

                subject = subject + "실행 성공 알림";
                text = text+ "job 실행이 성공 되었습니다";

            }

            messageHelper.setTo(""); // 전송할 이메일 주소 (받는 사람)
            messageHelper.setSubject(subject);
            messageHelper.setText(text);

            javaMailSender.send(message);


        }catch(Exception e){
            e.printStackTrace();
        }
    }
}


```
그리고 listner 을 작성해서 각각 필요한 job 이 실행되기전 메세지 만들어서 전송을 하고 , job 이 끝나고 job 의 성공 유무를 판단해서 또 한번 이메일을 작성을 하겠습니다 그러면 
결과는 이렇게 나오게 됩니다 


## 이메일 전송 결과
![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/8abb7fda-623e-429a-bae9-ec9ef80af5cb)

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/901111d0-5d2a-4ee9-b4ed-241520a0091d)

이렇게 나오는것을 확인할 수 있습니다 이렇게 우리는 Batch 의 실행 및 결과에 대해서 이메일로 받아볼 수 있게 진행을 했습니다 다음에는 slack 이라는것을 연동해서 슬랙bot 으로 
진행을 해서 결과를 알려주는 형식으로 진행하겠습니다