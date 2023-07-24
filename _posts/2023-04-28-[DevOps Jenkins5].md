---
title: DevOps Jenkins 처음으로 하는 jenkins 빌드
author: kimdongy1000
date: 2023-04-30 12:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 최소한의 패키지 설치로 jenkins 와 연동을 진행을 했고 이제는 외부 프로젝트를 가져와서 빌드를 진행을 해보겠습니다 

## 새로운 job 생성

이번에 할것은 새로운 gitlab 으로 프로젝트 연동하는 것입니다 마찬자기로 좌측에 있는 새로운 item 클릭후 

![화면 캡처 2023-07-19 141613](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/547211fa-13ff-4aa4-b36b-6202c87b8960)
타이틀은 이렇게 하고 두번쨰 Maven Project 를 클릭해줍니다 

아마 처음 jenkins 를 최소사양으로 설치하면 maven 의 플러그인이 없을것이 뻔하기에 
뒤로 돌아와서 좌측 메뉴의 jenkins 관리 -> Plugins 클릭합니다 

![화면 캡처 2023-07-19 141813](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7b750a90-a5f2-4285-8d72-6aee1b86db92)

그리고 이렇게 Maven Integration plugin 를 설치해줍니다 이 화면은 저는 이미 설치를 했기때문에 installed plugins 에 들어오게 됩니다 
최초 설치는 좌측 메뉴중에 Available plugins 를 클릭해서 설치를 진행하면됩니다 

저는 보통 github 보다는 gitlab 을 주로 많이 사용합니다 둘의 차이점은 거의 없지만 저는 gitlab 위주로 설명을 진행하도록 하겠습니다 
첫번째 설명에 본인이 원하는 이 job 의 설명을 적으시고 

하단 소스코드 관리로 와서 자신의 gitlab 또는 github 의 clone 주소를 입력합니다 
이때 조심할것은 외부로 공개된 프로젝트만 가능합니다 만약 현재 자신이 입력할 프로젝트가 비공개 상태라면 다음 장에 비공개 프로젝트 연동에 대해서 공부할것입니다 
이번장에는 공개된 프로젝트를 어떻게 연결해서 빌드를 하는지에 대해서 공부를 할것입니다 

그러면 이렇게 될것입니다 

![화면 캡처 2023-07-19 142324](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/d1479d8c-5ee4-4277-b9a1-35ff2cfdc0c1)

이런 모양이 될것이고 하단에 어떤 브랜치를 빌드할것인지 정하는데 보통 해당 프로젝트의 protect 브렌치를 많이 빌드합니다 안전하다고 생각하는 브렌치기 떄문에 
이건 선택이기 떄문이기에 최종 빌드되서 나갈 브랜치를 선택하면되는데 저는 main 으로 선택하겠습니다 

Branch Specifier (blank for 'any') */main 으로 입력하겠습니다 

빌드유발 빌드환경 이런것은 다 꺼주시고 

하단에 Build 에 오면 Root Pom 현재 우리가 선택한 job style 는 maven 스타일이기 때문에 pom.xml 위치를 물어보는 경우가 있습니다 이때는 프로젝트의 root 에 있으면 
그대로 주시면되고 만약 root 위치에 없으면 root 준으로 상대경로 입력해주시면됩니다 

그다음 post - step 가 보이는데 
이는 위에서 빌드의 성공여부로 다음 스텝을 나갈지 그만둘지 결정하는 단계입니다 

Run only if build succeeds 보통이것을 많이 사용합니다 뜻은 오로지 빌드가 성공하떄만 다음 스텝으로 넘어갑니다 
Run only if build succeeds or is unstable 빌드가 성공하거나 , 불안정한 경우만 
Run regardless of build result 빌드결과 무시 

그렇기에 보통은 제일 위에를 많이 선택합니다 

우리는 빌드가 성공하면 shell 에 echo 를 써줍니다 

![화면 캡처 2023-07-19 143852](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7c118ff4-98b0-4e02-bb1d-ecda24793e54)
이렇게 됩니다 


이렇게 하고 저장을 하겠습니다 

그리고 이제 지금빌드 버튼을 누르면 빌드가 시작됩니다 

그리고 해당 console.log 를 보고 계시면 
위에 막 뭐가 뜨고 마지막 

```

[INFO] --- install:3.1.0:install (default-install) @ project_jenkins_module1 ---
[INFO] Installing /var/lib/jenkins/workspace/Hello_Jenkins_Build_Project/pom.xml to /var/lib/jenkins/.m2/repository/com/kimdongy1000/time/project_jenkins_module1/1.0.0/project_jenkins_module1-1.0.0.pom
[INFO] Installing /var/lib/jenkins/workspace/Hello_Jenkins_Build_Project/target/project_jenkins_module1-1.0.0.jar to /var/lib/jenkins/.m2/repository/com/kimdongy1000/time/project_jenkins_module1/1.0.0/project_jenkins_module1-1.0.0.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  8.993 s
[INFO] Finished at: 2023-07-19T14:40:03+09:00
[INFO] ------------------------------------------------------------------------
[WARNING] 
[WARNING] Plugin validation issues were detected in 2 plugin(s)
[WARNING] 
[WARNING]  * org.apache.maven.plugins:maven-compiler-plugin:3.10.1
[WARNING]  * org.apache.maven.plugins:maven-resources-plugin:3.3.0
[WARNING] 
[WARNING] For more or less details, use 'maven.plugin.validation' property with one of the values (case insensitive): [BRIEF, DEFAULT, VERBOSE]
[WARNING] 
Waiting for Jenkins to finish collecting data
[JENKINS] Archiving /var/lib/jenkins/workspace/Hello_Jenkins_Build_Project/pom.xml to com.kimdongy1000.time/project_jenkins_module1/1.0.0/project_jenkins_module1-1.0.0.pom
[JENKINS] Archiving /var/lib/jenkins/workspace/Hello_Jenkins_Build_Project/target/project_jenkins_module1-1.0.0.jar to com.kimdongy1000.time/project_jenkins_module1/1.0.0/project_jenkins_module1-1.0.0.jar
[Hello_Jenkins_Build_Project] $ /bin/bash /tmp/jenkins14282609946754023288.sh
channel stopped
빌드성공
Finished: SUCCESS


```

이렇게 뜬것을 확인했습니다 우리는 그럼 처음으로 git 과연결해서 maven 프로젝트를 빌드해보았습니다 그리고 마지막 우리가 echo 에 넣은 빌드성공 문구 까지 보았습니다 
이게 하나의 빌드 사이클입니다 우리는 앞으로 이 내용을 가지고 정말 많은 예제를 해보면서 다양한 방식으로 프로젝트를 빌드해보겠습니다 

