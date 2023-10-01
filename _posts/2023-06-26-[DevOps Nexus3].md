---

title: DevOps Nexus 3 나만의 저장소 만들기
author: kimdongy1000
date: 2023-06-26 16:30
categories: [DevOps, Nexus]
tags: [ Nexus ]
math: true
mermaid: true

---

지난시간에는 nexus 를 걸치하면 기본적으로 제공하는 maven - central 에 대해서 공부를 했습니다 이번시간에는 나만의 연결소를 만들어서 이전시간에 중앙연결소와 같이 묶어서 
저의 로컬에 연결하는 작업을 진행하도록 하겠습니다 

## 그룹 만들기 

그룹은 여러개의 repository 를 하나의 그룹으로 묶어서 사용할 수 있는 단일의 repository 로 사용할 수 있습니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/12a662f6-6c15-4c46-b363-8d86095a231e)

1. 상단에 톱니바퀴를 누르고 좌측에 Repositories 를 클릭후 메뉴에 Create repository 를 클릭합니다 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/71398342-c631-4953-92c0-bf969810d904)

2. 중간에 나오는 것들 중에 maven2(group) 을 클릭합니다

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0f02dc49-73ba-449c-bd56-7f93ffb827f6)

3. 상단에 적절한 타이틀 주고 나머지 설정은 만지지 않은채로 

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0ba59863-0c88-4b03-a77e-d070a6dcde61)

4. 그룹으로 추가할 repository 를 오른쪽으로 옮겨줍니다 우리가 현재 사용하는 repository 는 오른쪽으로 옮겨줍니다 

5. 그리고 하단에 create repository 를 만들게 되면 생성이 되고

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/fc898f7b-6ea3-4cb0-81b7-333973f9724f)

6. 이렇게 생겨나게 됩니다 

이것이 그룹을 만드는것이고 

## 나만의 reposiroty 만들기 

1. 상단에 톱니바퀴를 누르고 좌측에 Repositories 를 클릭후 메뉴에 Create repository 를 클릭합니다 

2. 중간에 나오는 것들 중에 maven2(hosted) 을 클릭합니다 이때 group 하고 hosted 의 뜻의 의미를 알수 있을거라 생각합니다

![7](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/5f80949c-c96d-4590-b785-ea0aed9e21e6)

3. 적절한 타이틀 주고 다른설정은 만지지 않습니다 

![8](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/187a2b06-6e28-4e03-8a1b-1f79399f4683)

4. Deployment plicy 를 allow redeploy 로 솔종 

이 설정에 대해서는 설명이 필요한데 로컬에서 maven 생명주기로 deploy 를 실행하거나 jenkins 배포로 deploy 를 생명주기를 실행하게 되면 이때 nexus 에 올라갈지 
안올라갈지는 이 설정에 달려 있습니다 allow redeploy 하게 되면 기존에 만들거나 새로 생기는 lib 같은 경우 nexus 에 배포가 되게 됩니다 이에 대한것은 로컬 그리고 jenkins 로 테스트 해보는것으로 하겠습니다 

5. 하단에 create repository 를 클릭합니다 

## 나만의 repository 를 group 에 포함시키기 

![9](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/cb939525-4b5f-4486-a945-061108bf3c9f)

1. 상단 톱니바퀴 클릭 

2. 왼쪽 메뉴 repositories 클릭

3. 방금만든 group 찾아서 하단으로 내려간뒤 아까 만든 repository 를 사진처럼 포함시켜줍니다 

이렇게 해서 우리는 우리만의 group , repository 를 만들보는 작업을 했습니다 다음시간에는 왜 우리가 나만의 repository 를 만들었는지에 대해서 알아보도록 하겠습니다 









