---

title: DevOps Jenkins 디렉터리구조
author: kimdongy1000
date: 2023-05-02 15:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 비공개 되어 있는 프로젝트를 ssh-keygen 이라는 비대칭 암호키를 만들어서 자신이 소유하고 있는 프로젝트를 빌드하는 것을 해보았다 이번시간에는 본격적으로 빌드에 앞서서 
jenkins 의 디렉터리 구조에대해서 공부하는 시간을 가져볼려고 합니다 디렉터리 구조에 대해서는 크게 신경쓸 필요는 없지만 최소한 이정도는 저는 알아야 한다는 생각을 가지고 있기에 그에대한 공부를 진행할려고 합니다 

## Jenkins_Home 
Jenkins 의 Home 은 어딜까 보통 이런 부분은 수정을 잘하지 않기 때문에 /var/lib/jenkins 가 기본 root 위치입니다 
jenkins 페이지에서도 jenkins 관리 -> System 으로 들어오시면 제일 처음 맞이하는 문구가 홈디렉터리 해서 표기가 됩니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/c5de5741-e63e-4016-80d0-b977d87f03c9)

이렇게 말이죠 그럼 서버상에서 구조를 보도록 하겠습니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f1744d68-9192-4f23-9aea-7d1015ee4596)

그럼 디렉터리 특징별로 한번 살펴보도록 하겠습니다 

## jobs 
이 디렉터리는 우리가 jenkins 웹에서 만든 job 들이 모이는 곳입니다 하나를 열어보면 아시겠지만 해당 job 의 특징 및 설정정보를 담아놓게 되는데 
특히 특정 jobs 안에 config.xml 을 살펴보면 우리가 웹에서 등록한 정보들이 이렇게 xml 로 저장되는데 우리 일부분을 가져와보자 

```
<description>나의 첫번째 gitlab 프로젝트</description>

<userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://gitlab.com/kimdongy1000/jenkins_module1.git</url>
      </hudson.plugins.git.UserRemoteConfig>
</userRemoteConfigs>

<branches>
      <hudson.plugins.git.BranchSpec>
        <name>*/main</name>
      </hudson.plugins.git.BranchSpec>
</branches>

<hudson.tasks.Shell>
      <command>#!/bin/bash

echo &quot;빌드성공&quot;</command>
      <configuredLocalRules/>
</hudson.tasks.Shell>


```

전부다를 추출하지는 않았지만 우리가 지난시간에 쭈욱 해왔던 내용들이 이렇게 xml 로 설정이 담겨 있는것을 확인할 수 있습니다 

## build history 
/var/lib/jenkins/jobs/Hello_Jenkins_Build_Project/builds/1 

빌드 히스토리는 특정 job 의 builds 에 들어가보면 이렇게 첫번째 빌드이면 1이라는 디렉터리가 만들어지고 그 안에 그 당시 build 상세 설정및 
로그를 저장하고 있습니다 이러한 정보를 바탕으로 jenkins 웹은 정보를 읽어서 웹에 뿌려주는 역활을 진행하고 있습니다 

## logs 
사실 여기에 대한 정보는 나도 없도 파일을 열어보면 job 에대한 기초적인 build 발생시간을 적어둔 정도로 보이니 pass 
그리고 jenkins 의 전반적인 로그는 /var/log/jenkins 에 쌓이게 된다 

## plugins
여기에는 우리가 다운로드 및 설치한 플러그인이 모이는 곳이다 사실 인터넷이 되는 환경이라면 웹에서 필요한 라이브러리를 받을 수 있지만 종종 폐쇄망에는 
외부에서 받아서 가지고 들어와야 합니다 여기에 필요한 파일을 둔뒤 재기동을 하면 그때 이 plugins 를 읽어서 필요한 라이브러리를 사용할 수 있습니다 
역시나 버전 맞추는것이 힘들어서 폐쇄망 환경에는 보다 제한적으로 외부에서 가져온 파일을 설치 할 수 있습니다 

## Users 
여기는 jenkins 사용자들을 모아두는 곳입니다 지금은 한사람만 사용하지만 종종 여러사람이 사용하는 경우에는 해당 사용자의 프로필이 생겨나게 됩니다 

## workspace 
사실 위의 디렉터리 구조는 그렇게 크게 알 필요 없지만 지금 이 workspace 는 반드시 알아가는것이 좋습니다 jenkins 의 핵심 작업장소이고 
모든 빌드건들이 이곳에 모이게 됩니다 우리가 jenkins 에 빌드버튼을 누르는 순간 jenkins 는 필요한 소스를 이곳으로 가져와서 fetch - pull 을 해서 빌드를 진행하고 
사용자가 원하는 이후 작업을 하게 됩니다 

workspace 안의 디렉터리도 마찬가지로 job 의 이름을 짓는대로 생겨나게 되며 우리가 지난시간 maven 프로젝트를 빌드를 한 곳에 들어가면 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/99247d4f-8c07-4466-a5da-aec2efe51af4)
이렇게 maven 이 빌드되면 target 에 쌓이는것을 볼 수 있습니다 이곳은 jenkins 의 메인 작업 무대이며 우리는 이곳을 활용해서 앞으로 그리고 계속 작업을 해나갈 예정입니다 

오늘은 간단하게 jenkins 디렉터리 구조에 대해서 공부해보는 시간을 가져보았습니다