---

title: DevOps Jenkins WebHook3
author: kimdongy1000
date: 2023-05-11 09:23
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 WebHook 을 활성화 하고 특정 브렌치에서만 빌드 트리거가 동작할 수 있게 설정을 해보았다 이번시간에는 나도 현업에서 이란 방식을 사용하진 않았지만 
앞으로 해보고 싶은 배포방식을 한번 연구해서 이 글에 담기로 하였다 결론부터 말하자면 dev 로 push 가 일어나는것들은 개발서버로 main 으로 push 가 일어나는것들은 운영서버로 
오늘은 그런 서버를 직접 만드는게 아니라 디렉터리로 분리해서 진행을 하도록 하겠습니다 

![dev_prod](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f426a827-4303-40e8-b552-77be7bb421b4)

일단 서버를 크게 나누기 보다는 디렉토리를 나눠서 진행을 하기로 했습니다 차후에는 서버를 나눠서 개발 , 운영 직접 배포하는 것을 보여드리도록 하겠습니다 
그리고 기존 데이터를 다 지우고 쉘을 수정하도록 하겠습니다 









