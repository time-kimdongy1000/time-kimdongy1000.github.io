---
title: DevOps Jenkins Job 만들기
author: kimdongy1000
date: 2023-04-28 12:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true
---

## Jenkins 의 Job 만들기 
일단 다짜고짜 Job 이라는것을 만들어보자 이 Job 은 Jenkins 에서 작업 계획 1개를 뜻합니다 메인 메뉴에서 왼쪽 상단 메뉴를 보면 새로운 아이템이라는 메뉴가 있습니다 이를 클릭하면 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/e9375928-9d34-41a0-8229-47e206955a60)

여기에서 상단 텍스트는 해당 Job 의 이름을 입력하시면됩니다 그리고 하단 메뉴에서는 Free Style 를 선택하고 하단 OK 버튼을 누르면 

![Hello-Jenkins-Config-Jenkins-](https://github.com/SH-Yeon93/ImageStore/assets/58513678/349db3cf-f52d-4599-a051-7ad42bfc3a35)

이런창이 나올텐데 

## General 
이 Job 의 전반적인 설명및 설정을 하는 부분입니다 

## 소스코드 관리 
어디서 소스를 가져올것인지 적습니다 github , gitlab 을 주로 사용합니다 

## 빌드유발 
어떻게 빌드를 하는지에 대한 설정을 하는곳입니다 

## 빌드환경
빌드를 할때 WorkSpace 에 대한 상태를 설정할 수 있습니다 

## Build Steps
추가적인 빌드단계를 설정할 수 있습니다 

## 빌드 후 조치 
빌드가 끝이나면 어떻게 하는지 설정할 수 있습니다 

이런 큰 메뉴만 보면 무슨말인지 모르겠지만 우리는 이에 대해서 Job 만드는거 부터 배포 실행까지 전부 해볼것이다 일단 우리는 설명부분에 Hello Jenkins 로 입력하고 
저장을 하고 나오겠습니다 



![Hello-Jenkins-Jenkins-](https://github.com/SH-Yeon93/ImageStore/assets/58513678/108e6012-835d-48a6-be1c-c68ee8464b6d)

이렇게 나오는데 옆에 실행을 하겠습니다 실행은 지금보이는 좌측메뉴에 지금빌드를 클릭하면됩니다 그럼 이제 빌드가 되고 하단에 Build History 에 #1 번이 생기는데 이를 클릭하면 

![Hello-Jenkins-1-Jenkins-](https://github.com/SH-Yeon93/ImageStore/assets/58513678/6dd9f077-f4d6-494a-9254-5bd7a342f24c)


화면이 나올것이고 이 화면에서 좌측에 있는 Console Output 을 클릭하면 

![Hello-Jenkins-1-Console-Jenkins-](https://github.com/SH-Yeon93/ImageStore/assets/58513678/26c58f93-2821-4a30-b8b8-c9289f5ece27)

```

Started by user Time
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/Hello Jenkins
Finished: SUCCESS

```

이렇게 나오는데 우리는 여기서 Finished: SUCCESS 가 나왔다면 잘된것이다 우리는 job 을 만드는거 부터 해서 빌드 그리고 빌드된 결과를 보았다 우리는 앞으로 
gitlab 을 통해서 연동을 통해 원격서버에서 소스를 가져와서 jar , 또는 war 로 말아서 원격서버에 배포하는것을 해볼것이다 그리고 위에서 설명만 하고 넘어갔던 
큰메뉴들을 하나씩 설정하면서 CI/CD 가 무엇인지 배울것입니다