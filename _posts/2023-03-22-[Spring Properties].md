---
title: Spring 기동시 다른 환경설정 주입하기
author: kimdongy1000
date: 2023-03-22 11:07
categories: [Back-end, Spring - Core]
tags: [ Properties ]
math: true
mermaid: true
---

최근에 어플리케이션 보안에 관련해서 생각하게 되었다 단순 코드를 보호하는것도 보안이지만 나의 설정파일을 외부에 알리지 않는것도 보안이라고 생각한다 
이번시간에는 서버 환경마다 다르게 properties 를 구성해서 이를 활용할것이다 

## Properties 

우리가 Properties 에 정말 다양한 정보를 넣게 된다 그중에는 서버 정보를 넣게 되는데 이런 민감정보를 누구나 열람할 수 있는 gitlab 같은곳에 올리게 되면 문제가 생길 수 있다 
그렇기에 이번기회에 이를 분리해서 개발을 진행할것이다 

## spring.config.location
이는 보통 잘 안쓰는데 이를 쓸때는 어플리케이션 내부에 프러퍼티 파일이 있는 경우이다 예를 들어서 어플리케이션 내부란 
src 아래에 있는 모든것들을 프로젝트와 관련이 있는 소스들이다 우리는 보안을 생각해서 이제 이곳에는 아예 프러퍼티를 두지 않을것인데 예전같았으면 
프러퍼티는 같이 안에 두고 같이 올리는 편이였지만 이제는 상황도 많이 바뀌고 시대가 변했다 지금부터는 properties 도 보안의 대상이 된다 
그래서 우리는 지금 보이는 프러퍼티는 외부 파일을 바라보게끔만 설정을 해둘것이다 

## spring.config.import
```
spring.config.import=file:/C:/webProject/20230522_properties/application.properties,file:/C:/webProject/20230522_properties/db.properties

```
현재 내가 가지고 있는 properties 의 파일이다 이 파일은 어디 파일을 바라볼뿐 아무것도 쓰지 않을것이다 원래 같았으면 이곳에 서버 설정 및 DB 설정같은 각종 설정을 해두지만 
이번 프로젝트는 보안을 생각해서 완전히 분리해서 진행을 할것이다 

그리고 서버의 구동환경에 따라서 properties 를 다르게 설정할것인데 결국 이 어플리케이션은 리눅스 위에서 동작이 될것이다 그래서 윈도우 환경과 다른 곳의 properties 를 하나 더 만들것인데 그전에 window 환경에서 properties 를 사용할려면 지금처럼 가지고 있는 properties 를 다음처럼 이름을 변경할것이다 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/d58f4fd7-560e-4d0a-ae8e-7cf30c48e7d7)

우리는 앞으로 properties 이름을 applition-dev.properties 로 둘것이다 그러면 이 프러퍼티는 우리가 실행변수(현재 사진 가운데 VM 옵션) 이렇게 주게 되면 
이 어플리케이션은 dev 이름을 가진 properties 를 로드하게 된다 마찬가지로 운영으로 넘어가게 되면 우리는 이 이름을 application-prod.properties 로 두고 기동을 하게 되면 
이제 prod 환경으로 파일을 읽게 되는것이다 

자 그럼 보안에 강력한 개발을 진행을 해보자 





