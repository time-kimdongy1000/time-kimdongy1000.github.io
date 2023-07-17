---
title: DevOps Jenkins 기타 소프트웨어 설치 및 연동
author: kimdongy1000
date: 2023-04-29 12:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true
---

사실 jenkins 프로그램 하나만 있다고 해서 바로 프로젝트 빌드를 할 수 없다 기본적으로 jenkins 웹을 띄울때만 사용되는 jdk 만 깔았을 뿐 다른 패키지도 깔고 연동을 해야 한다 
사실 yum 으로 설치하고 주소 연동하면되지만 jenkins 에서는 내부에서 필요한것들을 설치 할 수 잇게 도와주는데 우리가 더 깔아야 할 패키지는 다음과 같다 
git , maven 은 필수로 설치를 해야 하고 nexus 는 당분간 central nexus 를 사용할것이고 그 외에는 자체구축해서 진행을 할것이다 

그래도 yum 으로 설치를 진행하도록 하겠습니다 

## git 설치
```

yum install -y git

```
사실 패키지 설치하는건 쉽가 다음과 같이 진행을 하면된다 그리고 마찬가지로 다른곳에서도 쓸 수 있게 path 를 잡아주자 

![2](https://github.com/SH-Yeon93/ImageStore/assets/58513678/86d3579c-5bb5-488e-96de-84ea57015ab2)

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/e3ae5f76-30e8-44a4-92fa-91cae5ae3315)

올바르게 설치되었는지 아닌지는 이 위치로 들어와서 이 부분에 빨간색 부분이 없으면된다 예를 들어서 git 을 설치했는데 /usr/bin/git 에 git 이 안잡혀 있으면 빨간색 에러가 날것이다 
이는 git 이 설치되었으면 자동으로 감지해서 위치를 잡는다 

## MAVEN 설치 
maven 에 대해서 잠깐 설명하면 기본적으로 java 로만 이루어저 있는 프로젝트 경우는 jdk 로만 빌드가 가능하지만 프로젝트가 점점커지면 이에 대한 부분에 대한 빌드가 필요하다 
기본적인 java 소스는 jdk 가 빌드하지만 외부 라이브러리 관리 및 기동가능한 상태로 패키징 가능하게 한 패키지는 바로 Maven 이다 이 또한 설치를 해야 하는데 
예는 버전마다 빌드 할 수 있는 java 버전이 다르므로 되도록이면 3.6 버전 이상 설치가 필요하다 

yum 으로 설치하면 3.0 버전밖에 설치가 안되므로 이 버전은 특정 jdk 버전 이상은 빌드가 불가능하다 그래서 우리는 최소 3.6버전 이상을 설치할것인데 
사이트에 나와있는 3.9버전으로 설치를 해보자 

![3](https://github.com/SH-Yeon93/ImageStore/assets/58513678/0f251371-cccf-464b-90a1-41039dcdc9a5)

해당링크 클릭해서 다운받는 알집으로 위치는 크게 상관 없지만 관리할 수 있는 곳에 옮겨둔뒤 우리는 알집을 해제 할 것이다 

이때 zip 파일을 압축하고 푸는 패키지는 각각 zip , unzip 이다 이를 알아서 설치한뒤 다음과 같은 명령어로 풀어주자 

```

unzip apache-maven-3.9.2-bin.zip

```

이렇게 하면 해당 zip 파일이 풀리게 된다 이 maven 은 jenkins 가 바로 인식을 못하니 환경설정후 연동을 해주어야 한다 

vi /etc/profile 
```

export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.el7_9.x86_64
export MAVEN_HOME=/usr/local/maven/apache-maven-3.9.2
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin


```
명령어를 통해서 기존의 jdk 과 연동해서 path 를 이어 붙인 다음 

저장후 나와서 

source /etc/profile 를 하면된다 

```

[root@localhost ~]# mvn -version
Apache Maven 3.9.2 (c9616018c7a021c1c39be70fb2843d6f5f9b8a1c)
Maven home: /usr/local/maven/apache-maven-3.9.2
Java version: 11.0.19, vendor: Red Hat, Inc., runtime: /usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.el7_9.x86_64
Default locale: ko_KR, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-1160.el7.x86_64", arch: "amd64", family: "unix"


```

그리고 다음의 명령어로 입력을 했을대 이렇게 나오게 되면 올바르게 설치된것이고 이를 jenkins 에 연동을 해야 한다 

![3](https://github.com/SH-Yeon93/ImageStore/assets/58513678/61e3c1ec-2260-4500-acd6-5b96745f98bc)

git 설치했을 제일 하단에 보면 Maven 이 있는데 여기에 이렇게 쓰면된다 연동 끝이다 그리고 저장을 하자 
이렇게 되면 일단 최소한의 패키지로 maven 프로젝트를 빌드할 수 있는 환경은 되었다 앞으로는 이를 통해서 실제 gitlab 하고 연동을 해서 소스를 fetch - pull 한 다음 빌드 진행을 하도록 할것이다 










