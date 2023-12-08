---
title: DevOps Jenkins 설치
author: kimdongy1000
date: 2023-04-27 11:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true
---

## Jenkins 의 시작

내가 왜 이것을 시작했는지 모르겠다 어느순간부터 나는 하나의 프로젝트에서 DevOps 를 맡게 되었다 처음에는 아무것도 몰랐다 그저 개발만 할줄알았던 나는 처음 여의도 모 프로젝트에 사수도 없는 프로젝트에서 홀로 DevOps 구축했다 DevOps 라고 하면 뭐 다들 대단하다고 생각하는데 그냥 개발 + 운영이 포함된 단어이다 이 운영은 개발에서 개발한것을 운영에 지속적으로 배포 및 유지보수 할 수 있는 시스템을 뜻하고 여기에는 대표적으로 Git , Jenkins , Nexus 가 있다 물론 요즘으 Docker 컨테이너 안에 이 3개를 전부 들고 다니면어 어디든 펼쳐놓고 쓰면되지만 
내가 한 프로젝트는 오로지 물리서버에 Git , Jenkins , Neuxs 등 소프트웨어가 깔리면 나는 해당 서버들 (개발 , 운영)  에 DevOps 를 연결해서 지속적인 배포 및 개발자들이 개발을 잘 그리고 유지할 수 있도록 서포트한다 물론 개발도 한다 그러다 보니 시간도 없는데 이것저것 하다 보니 전문성도 좀 떨어질 수 있지만 최근에는 Jenkins 쪽을 상당히 그리고 Nexus 도 상당히 많이 공부했다 이번 챕터에서는 Jenkins 의 설치부터 활용까지 해볼 정이다 원래 설치는 나의 계획상에는 없었지만 되짚어본다는 마인드로 설치부터 이제까지 내가 Jenkins 를 구축해 오면서 
어려웠던 점 힘들었던 점 전부를 공유할 예정이다 

## 환경 
나는 리눅스 시스템에서 이를 관리해보았기에 우리는 Centos7 에서 Jenkins 를 해볼것이다 리눅스를 설치해보지 않은 사람은 한번 구글링에 따라 설치를 해보면 될것이다 
생각보다 어렵지 않고 쉬운 구성으로 설치가 가능하다 

## 설치 
Jenkins 를 설치하기위해선 JDK 를 설치해야 한다 Jenkins 가 java - spring 기반 웹으로 만들어졌기 때문에다 그래서 jenkins 를 기동하기 위해선 8버전 이상의 Jdk 가 필요한데 우리는 
11버전 으로 깔것이다 

yum list java*jdk-devel

```

java-1.6.0-openjdk-devel.x86_64                    
java-1.7.0-openjdk-devel.x86_64                    
java-1.8.0-openjdk-devel.i686                      
java-1.8.0-openjdk-devel.x86_64                    
java-11-openjdk-devel.i686                         
java-11-openjdk-devel.x86_64                       


```

현재 yum 에서 설치할 수 있는 버전으로 8버전과 11버전이 있다 우리는 11버전으로 설치할것이다 

yum install -y java-11-openjdk-devel.x86_64  


## 환경설정 

```
which java  를 하면 심볼릭 링크가 나오게 되는데 우리는 원본파일의 위치가 필요함으로 

readlink -f /bin/java

/usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.el7_9.x86_64/bin/java 이렇게 심볼릭 링크가 어디를 가리키는지 볼 수 있다 

이를 /etc/profile 에 등록을 해야 한다 

vi /etc/profile 

export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.el7_9.x86_64
export PATH=$PATH:$JAVA_HOME/bin

```


저장하고 나오고 

```
[root@localhost ~]# source /etc/profile
[root@localhost ~]# $JAVA_HOME
-bash: /usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.el7_9.x86_64: 디렉터리입니다
[root@localhost ~]# java -version
openjdk version "11.0.19" 2023-04-18 LTS
OpenJDK Runtime Environment (Red_Hat-11.0.19.0.7-1.el7_9) (build 11.0.19+7-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-11.0.19.0.7-1.el7_9) (build 11.0.19+7-LTS, mixed mode, sharing)
```


현재 세션에서 바로 저장하는 source /etc/profile 저장해서 나온뒤 $JAVA_HOME 했을때 설치한 java Path 가 나오면 잘 설치된것이다 

jenkins 를 설치할려면 다른 리포지토리에서 가져와야 한다 그래서 

```
 sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
  sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

```

이 명령어를 가져와서 실행하면 이제 jenkins 를 설치할 수 있다 이게 key 가 가끔 바뀔수가 있어서 공식에서 확인하길 바랍니다 
https://get.jenkins.io/redhat-stable/

## 실행 
systemctl start jenkins 
예전엔 service jenkins start 였지만 어느 순간부터 systemctl start jenkins 써야 실행이 될것이다 

## 방화벽 해제 
jenkins 의 방화벽은 8080 임으로 이 8080 포트를 방화벽 해제할것이다 

firewall-cmd --permanent --zone=public --add-port=8080/tcp

그리고 자신의 host 주소로 접속을 하게 되면 이제 jenkins 웹화면으로 나올것이다 그럼 다음의 화면이 나올것이다 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/2d11d7c8-77a0-4606-b41c-2148aaee3607)

이런 화면을 맞이할것인데 vi 로 해당 텍스트 열어서 입력해주면 된다 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/f5424c30-c4a1-4de9-884a-4c49c2801fcd)

그리고 우리는 기본모드로 설치 그러면 기본으로 필수적으로 설치해야할 플러그인을 설치할텐에 이 부분이 좀 시간이 걸린다 
다만 이 부분에서 설치가 안되는것들은 있을텐데 그것은 JDK 버전 또는 Jenkins 버전이 해당 플러그인과 맞지 않아서 발생하는것이다 그건 넘어가서 jenkins 버전을 업데이트 할 수 있으니 
이 화면에서는 그저 기다리기만 하면된다 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/994d165c-aaab-44a6-a829-c16b4c5aaeed)

그리고 설치가 끝나면 계정을 생성하는데 여기서 쓰는 계정은 이제 관리자 계정이 됨으로 잘 기억하고 사용하자 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/a5fc303a-29b4-4711-8e9c-963fa9106e61)
그렇게 해서 로그인을 성공하면 이와 같은 화면이 나온다 jenkins 도 예전것을 보면 UI 가 상당히 구리지만 요즘은 Blue Ocence 를 개발하면서 상당히 세련된 UI 를 제공하고 있다 
우리는 최신 버전에서 Jenkins 와 관련한 다양한 CI/CD 지속적인 배포에 대해서 공부를 해볼것이다