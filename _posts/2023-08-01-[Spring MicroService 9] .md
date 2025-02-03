---
title: Spring MicroService 9 Spring MicroService Cloud-Config Github
author: kimdongy1000
date: 2023-08-01 14:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

지난시간까지는 단순 파일 Path 로만 이용해서 컨피그 서버를 구축하고 만들었습니다 이번시간에는 GitHub 를 이용해서 컨피그 서버를 연동해서 사용하는 방법으로 
컨피그 서버에 대한 정리는 끝내도록 하겠습니다 

자 그럼 간단하게 git을 먼저 만들어보자

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ea713cb3-0cd4-4397-b31f-bdaf1bdb9d55)

먼저 public 한 GithubRepository 를 먼저 만들겠습니다 이곳에서는 우리가 앞에서 파일Path 로 이용한 컨피그들을 이곳에서 저장을 하고 사용할 예정입니다
전체코드는 지난시간 @RefreshScope 하고 동일하기 때문에 바뀌는 부분에 대해서만 정리를 하겠습니다

## config-server application.yml

```
spring:
  application:
    name: config-server

  profiles:
    active: git

  cloud:
    config:
      server:
        git :
          uri : https://github.com/time-kimdongy1000/config-server
          default-label: main



server:
  port: 8087

```

컨피그서버는 마찬가기로 application.yml 에 다음과 같은 정보를 기록합니다 이때 active 는 git 으로 두고 git 의 uri 는 만들어둔 컨피그 서버 주소를 담아둡니다 그리고 default-label 은 기본적인 브랜치를 뜻합니다 

## github - config-server-dev.yml
```

spring:
  datasource:
    hikari:
      url : jdbc:mysql://localhost:3308/dev_database
      username : dev_user
      password : dev_password
      driver-class-name : com.mysql.cj.jdbc.Driver



  config :
    value1 : test100
    value2 : test2

```
그리고 이 파일은 github 에 저장된 컨피그 파일 일부분입니다 지난시간 file 과 동일한 컨피그를 넣어두었습니다 그리고 컨피그 클라이언트는 지난시간하고 똑같기 떄문에 따로 올리지는 않겠습니다


## 기동
기동하게 되면 이전시간하고 별 다를게 없어보입니다 똑같이 클라이언트는 서버주소를 읽게 되고 서버는 클라이언트의 요청에 따라서 설정정보를 내려줍니다

## post-man 기동

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/2378ff27-2e30-46ca-a2ce-8b5476b1b240)

아마 이 부분까지는 동일할것입니다

## github - config-server-dev.yml 수정

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7ee4966f-e75b-40c3-8188-ea73ff6e7bde)

github 웹 에디터 상으로 수정을 하고 push 를 하게 됩니다

## config-client /actuator/refresh 요청

이 요청을 넣게 되면 config-client 는 서버에 요청을 넣어서 설정파일이 변경되었는지 확인을 하게 됩니다

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/d5ca610f-9a8f-4f2b-bab9-8bf686076864)

그 결과 다음과 같은 변경점이 return 이 되어서 돌아오게 되고

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/c09af548-db42-48cd-b648-fcf9fcc32a88)

이렇게 우리는 간단하게 config-server github 에 있는 정보를 갱신해서 가져오는 방법을 보았습니다 그럼 앞에서 했던 파일 시스템과 github 방법을 비교해보겠습니다


## 문제점
현재 git은 public 하게 되어 있어서 uri만 알게 되면 누구든지 들어올 수 있습니다 그래서 우리는 private 한 저장소를 만들어서 연동하는 방법을 바로 진행을 하겠습니다

## GitHub Private 로 변경

![1](https://github.com/user-attachments/assets/06f0623c-f92c-47c1-9b0c-80c7d0bffb4c)

먼저 세팅으로 가서 아래 버튼을 눌러서 private 프로젝트로 만들어줍니다 그러면 private 할 시 다음의 문제점이 발생할 수도 있습니다 하면서 이를 이해 했으면 이 버튼을 눌러주세요 할때 누르면 리포지 토리가 private 하게 변경이 됩니다 

## 아무런 설정 없이 클라이언트 서버 재기동

```
 Caused by: org.eclipse.jgit.errors.TransportException: https://github.com/time-kimdongy1000/config-server: Authentication is required but no CredentialsProvider has been registered         

```

그러면 서버측에서 미친듯한 에러가 발생합니다 이유는 즉슨 Authentication 없다는 것입니다 그러면 우리는 이 Authentication 을 주는 방법 2가지를 만들어보겠습니다

## 아이디 비밀번호 방식
이 방식은 해당 private repository 에 접근이 가능한 계정의 아이디와 비밀번호를 서버 계정에 입력을 해두는 것입니다
애 username 은 github username 이고 password 는 실제 로그인 비밀번호가 아닌 토큰을 발급받아서 사용해야 합니다 토큰은 이와 같이 발급이 가능합니다

## github 토큰 발급

![1](https://github.com/user-attachments/assets/ec0bc06d-3de2-467c-9aae-58b3dde0e348)

이렇게 토큰을 발급해주면됩니다 특별한 조건 없이 그냥 public 한 리포와 private 한 리포를 읽을 수 있는 토큰으로 만들어주시면됩니다 이렇게 하면 이제 private 한 repository 도 접근이 가능합니다 

## config-server application.yml
```

spring:
  application:
    name: config-server

  profiles:
    active: git

  cloud:
    config:
      server:
        git:
          uri : https://github.com/time-kimdongy1000/config-server
          default-label: main
          username: 
          password: 
server:
  port: 8087


```
컨피그 서버에 이와 같이 username 과 passwrod 를 넣어서 연동을 해주시면됩니다 그리고 마찬가지도 기동을 하게되면 이제 인증 이슈는 없어지게 됩니다 

## id_rsa 방식
이 방식은 아마 알고 있을 방식중에 한 가지입니다 다른 서드파티 어플리케이션에서 git을 연동한적이 있다면 다들 아시는 방법입니다 이는 윈도우에서 id_rsa 를 만들게 되면
2개의 key 가 만들어집니다 하나는 공개키 , 하나는 개인키입니다 이 공개키를 git 서버에 등록을 해주고 개인키는 스프링에 연동을 해주시면됩니다

## key 생성 - window 
![1](https://github.com/user-attachments/assets/53299123-ffa0-4fb1-8b6d-6f08e3b7f383)

cmd 에서 `ssh-keygen -t rsa -b 4096 -C "kimdongy1000@gmail.com"` 이라고 입력을 하게 됩니다 이때 뒤에 있는 gmail 은 자신의 github 계정을 입력해주세요 그러면 그 페이지에서 해당 key 가 생성이 되게 됩니다 이때 2개의 파일 id_rsa , id_rsa.pub 이 만들어집니다 pub 파일을 github 에 등록을 해줍니다 이때

![1](https://github.com/user-attachments/assets/1ead710d-d247-4e60-848d-c543a5a735c3)
이곳에 말이죠

그리고 id_rsa 는 config 서버에 등록을 해줍니다 




```

spring:
  application:
    name: config-server

  profiles:
    active: git

  cloud:
    config:
      server:
        git:
          uri : https://github.com/time-kimdongy1000/config-server
          default-label: main
          ignore-local-ssh-settings: true
          privateKey:
            -----BEGIN OPENSSH PRIVATE KEY-----
            ...

            Vj3ZCVb7/TY7+yOR9SkHv0YB6uvMdivoZnnSTjD01k+xk8VqoseB8zqBLztvt0VE+6Rg6u
            oX698vph0+MouMx3hP3fPxl6TUH/PDGFbZQCIu+Fy9Hz3wr1HLGSsSla+QEBnUaKB7dgL9
            BJy/zRXYHBZ2HVjRnDhmexqiRZP+T1b3KtIoRsRLkwvuiBsqZMfMTvRmJJdGshtH6rm526
            1vj2ZWbrqk1WVVyNiWCnWcPmki2McIFIY/YfHlKkMzggYnKnmhGOxv+sGiwrtF2PmBNa/N
            0dNDfzA+1KjgSwyzDf/gSoH4utJv6SDc2EVAKEzHTCSpFJKH5c3TvVqJs7eZSjOO+OPNPo
            5tk8RRLbbloHqUi7FDhBeQ8TpC02qziq1+BnfgcnAAAFgN28S4rdvEuKAAAAB3NzaC1yc2

            .....
            -----END OPENSSH PRIVATE KEY-----





server:
  port: 8087

```

이렇게 안에 있는 모든 내용을 복사해서 붙여넣어줍니다 이때 필수로 `ignore-local-ssh-settings: true` 그렇지 않으면 window 의 ssh 정보를 읽게 됩니다 이걸로 30분은 허비했네요
그리고 기동을 하면 마찬가지로 서버와 연동된채로 사용이 가능합니다 

이렇게 해서 우리는 public 한 git repo 를 구축해서 컨피그를 연동해보았고 보안에 좀더 힘을 써서 private 한 방식으로도 연동을 해보았습니다 



## Git과 연동한 구성 파일 관리의 장점

```
1. 버전 관리
Git의 강력한 버전 관리 기능을 통해 구성 파일의 변경 이력을 추적할 수 있습니다. 누가 언제 무엇을 변경했는지 쉽게 확인할 수 있으며, 필요시 이전 버전으로 롤백할 수 있습니다.

2. 협업
Git을 사용하면 여러 개발자가 동시에 구성 파일을 수정하고 변경 사항을 병합할 수 있습니다. 이는 팀 간 협업을 크게 향상시킵니다.

3. 보안
Git 리포지토리의 접근 권한을 제어하여 구성 파일의 보안을 강화할 수 있습니다. 특정 사용자나 팀만이 구성 파일을 수정할 수 있게 할 수 있습니다.


```
지급 보면 이것은 일반 형상관리 특징하고 크게 다를것이 없습니다 맞습니다 config-server 를 github 를 사용하게 되면 git 의 장점을 모두 녹여낼 수 있습니다

## Git과 연동한 구성 파일 관리의 단점

```

1. 복잡성
Git과 Spring Cloud Config Server를 설정하고 관리하는 데 추가적인 학습과 노력이 필요합니다. 특히 작은 프로젝트에서는 복잡도가 높아질 수 있습니다.

2. 의존성
애플리케이션이 Git 서버와의 연결 상태에 의존하게 됩니다. Git 서버가 다운되거나 네트워크 문제로 인해 접근할 수 없는 경우 구성 파일을 가져오는 데 문제가 발생할 수 있습니다.

```
이렇게 우리는 간단하게 github 를 이용해서 컨피그 서버를 연동을 해보았습니다 
