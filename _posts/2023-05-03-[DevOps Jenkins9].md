---

title: DevOps Jenkins WebHook
author: kimdongy1000
date: 2023-05-03 15:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

영어 Hook 이라는 단어를 알고 있을까? Hook 낚시바늘이라는 뜻이다  React 에서 한번 들어본적이 있다 상태변화 관리에서 Hook 을 사용하며 특정 상태의 변경을 감지하는것을 뜻한다 이때 중요한것은 상태변화라는 것이다 jenkins 에도 Hook 이 있다 webHook 이라는 것이 있는데 이는 gitlab 의 상태를 jenkins 가 주시하고 있다가 변화가 생기면 
이때 변화는 (소스코드를 포함한 다양한 것을 말합니다) 자동으로 빌드작업을 시작한다 

오늘은 이 WebHook 을 해보겠습니다 

## 주의 
webHook 은 로컬 네트워크를 사용하고 있는 곳에서는 public 아이피를 받아서 포트포워딩을 해주셔야 합니다 이때 private IP 는 192.168 로 시작하게 되는데 이는 사설 아이피
같은 인터넷 망 안에서 서로다른 기기를 연결할때 발급해주는 ip 이고 공용 아이피는 실제 다른 외부 사람의 나의 페이지에 접속할대 사용하는 주소를 말합니다 
그래서 private 주소를 사용하는 곳에서는 공용ip 를 발급받아서 사용을 해야 한다 그래서 저는 vm 을 하나 더 설치해서 그 안에 gitlab 을 설치형 gitlab 을 구성하고 
webHook 을 만들어보겠습니다 이번시간에는 gitlab 설치에 관한건 다루지 않습니다 


## 사설아이피 허용

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/77a004b6-a199-4b75-89dc-4954b55ac8b0)
먼저 gitlab 의 어드민 권한으로 Allow requests to the local network from webhooks and integrations 를 체크해줍니다 이 체크를 통해서 내부 사설 아이피의 webhook 요청을 허용하는 것입니다 

## jenkins 에서 webHook 설정

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/c84a62e3-7f69-421f-bbcd-d8f7c4396086)

먼저 gitlab 주소를 변경을 해줍니다 이번시간에는 gitlab 설치에 대한것은 다루지 않기에 이에 대하서는 넘어가고 설치가 된 다음 프로젝트를 하나 생성해서 이전 작업까지 전부 완료를 해서 배포가 되는것을 먼저 확인을 합니다 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/2eedc65f-2f3f-4770-8e83-8cce4feb98e9)

이 부분이 실제 webHook 설정입니다 빌드유발에서 
Build when a change is pushed to GitLab. GitLab webhook URL: http://192.168.46.129:8080/project/Hello_jenkins_build_project2

Build when a change is pushed to GitLab gitlab 에서 push 가 될때 빌드를 해라이고 이때 주소는 
http://192.168.46.129:8080/project/Hello_jenkins_build_project2 가 될것입니다 이 주소는 gitlab 에 입력할 주소입니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/3716cbd2-ff9d-492e-95bc-800defe7a6e9)
그리고 하단 고급을 눌러서 Secret token 을 발급해줍니다 gitlab 이 jenkins 에 대한 자동빌드를 허락했다는 토큰입니다 물론 노출되어서는 안되지만 저는 사설망의 설치형 vm 이기 떄문에 문제가 되진 않습니다 

## gitlab 에서 webhook 설정 
![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/9549ef96-516a-415a-b3b3-bab0f7d59883)

gitlab 에서 webhook 설치는 프로젝트 하단 setting 에 보면 하위 메뉴로 등록이 되어 있습니다 저는 이미 하나 만들어서 테스트를 완료 했습니다 그래서 이미 하나가 있는것입니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/8ed8060a-ee00-4043-b386-813d78f9f7cb)
지금 보면 URL 엔 아까 jenkins 에서 주었던 주소를 입력해주시고 Secret token 고급을 눌러서 생성한 토큰을 입력해줍니다 
그리고 트리거는 빌드가 발생할 조건을 뜻하게 되는데 이번시간에는 webHook 이 동작되는것을 보기만 하면되기 때문에 All branches 를 선택하고 진행을 하겠습니다
다음시간에는 좀더 정교하게 동작하는것을 보겠습니다 

일단 이러고 저장을 하게 되면 이렇게 생깁니다 이때 에러가 발생하면서 local network 가 발생한 분들은 위의 사설 ip 에서 허용하는 것을 진행하셔야 합니다 

## WebHook 동작시키키 
동작은 간편합니가 제가 소스를 수정하고 commit push 하면 자동으로 됩니다 
여기서는 동영상이 없어서 체감은 못하겟지만 소스가 변경되자 마자 자동으로 설정한 job 이 실생이 되는것을 볼 수 있습니다 그렇다고 구분이 안되는것은 아닙니다 


![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/390ea044-ce8a-4c25-a42d-8de7d5402785)
![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ba8e0673-4b31-46c3-bfb5-776bee0cf5bc)

일단 이 둘의 시간대를 보면 거의 동시입니다 물론 push 되자마다 누른거 아니야 할 수 있겠지만 아래 사진을 보면 수행하는 자가 다르게 보입니다 

Started by GitLab push by kimdongy1000 gitlab 의 push 로 시작이 되었다 우리 평소의 다른 기록을 보면

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/6b22406d-7ac4-4f32-b5f7-1b1c5c02a2b0)
이렇게 아무런 그게 없습니다 그와는 달리 gitlab 의해서 시작된것을 볼 수 있는것입니다 

오늘은 이렇게 소스가 변경되는것을 감지해서 자동으로 build 되는 webHook 에 대해서 공부해보았습니다 다음시간에는 어디까지 변경했을때 감지를 하는지 그에 대해서 
공부를 하겠습니다 

