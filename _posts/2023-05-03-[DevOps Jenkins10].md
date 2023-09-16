---

title: DevOps Jenkins WebHook2
author: kimdongy1000
date: 2023-05-04 15:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---


지난시간에 우리는 webHook 을 연결해서 jenkins 가 gitlab 소스 변화를 감지해서 자동으로 빌드하는것에 대해서 알아보았다 오늘은 좀더 다양한 방법의 트리거에 대해서 알아보도록하겠습니다

다만 지난시간의 다 한것처럼 보이지만 현재 webHook 의 설정에는 문제가 있다 예를 들어서 현업에서 소스는 다음처럼 관리되지 않는다 보통 하나의 브렌치를 main 으로 두고 그 
아래 무수히 많은 자신만의 브렌치를 가지고 merge - request 를 통해서 배포가 될 브렌치에 병합을 하게 된다 다만 현재 이런 구조로는 아무 브렌치나 전부 배포가 되게 된다 


## 현재 설정의 문제점 
기본적인 설정만 하게되면 모든 브랜치의 push 건에 대해서 트리거가 발생하게 됩니다 예를 들어서 지금 하나 브렌치를 만들고 소스를 변경하고 push 를 해보자 

![스크린샷 2023-08-06 105416](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/6e563e2b-470c-49f3-ac04-f30a523ca78a) 
지금보면 메인에 push 하지도 않고도 빌드가 되는것을 확인할 수 있다 그리고 배포되는것을 확인해도 무단으로 배포된것을 확인할 수 있다 

이런 무분별한 브랜치에 대한 배포를 막기 위해서 우리는 webHook 의 특별한 설정을 해주어야 한다 

![스크린샷 2023-08-06 105416](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/330bd64e-0077-473c-8891-9afdaa39db84)
이곳을 보면 특정 브랜치만 트리거를 동작시킬 수 있는 설정이 있다 예를 들어서 main 브랜치에 변화가 있을때만 설정하고 싶으면 

Target Branch Regex 에 main 이라고 입력을 하면된다 그렇게 되면 main 에 대한 소스 변화가 없으면 이 job 은 트리거가 되지 않게 되는데 저장후 다시 소스를 수정을 하고 자신의 원격 브랜치에 push 를 해도 트리거가 동작이 되지 않는다 
이때는 우리는 main 에 변화를 주자 메인 소스 변화는 올라온 merge request 에 대한 승인을 해주면된다 그럼 머지를 하자마자 바로 jenkins 는 빌드를 시작하게 된다 이렇게 특정 브랜치 상태에서만 job 트리거를 활성화 할 수 있는데 
조금 재미있는 방식으로 가보자 

## main , dev 원격 브랜치 생성 및 트리거 활성화 
우리는 앞으로 main , dev 원격브랜치에 대한 트리거를 활성화 시킬것입니다 

![스크린샷 2023-08-06 105416](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/361b1b9b-b9e3-4c3b-b039-f72b2c14a392)

그리고 Target Branch Regex (main|dev) 설정을 해두게 되면 앞으로 main , dev 브랜치를 제외하고는 이 트리거를 활성하 할 수 없습니다 다른 여타 비슷한 이름 main2 , dev2 로 push 해도 이 트리거는 동작하지 않게됩니다 

즉 이 설정을 통해서 다른 브랜치가 올라오더라도 트리거가 동작하지 않는 설정을 해두었습니다 이 설정을 통해서 앞으로 좀더 재미있는 
빌드 환경을 만들어볼 예정입니다 








