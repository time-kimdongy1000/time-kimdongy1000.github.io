---

title: DevOps Nexus 1 Nexus 와 설치
author: kimdongy1000
date: 2023-06-20 09:23
categories: [DevOps, Nexus]
tags: [ Nexus ]
math: true
mermaid: true

---

아마 Nexus 에 대한것은 잘 모를 수도 있다 하지만 DevOps 의 부분중에 형상관리 GIT , 지속적 배포 Jenkins , 라이브러리 관리 Nexus 가 있습니다 
Nexus 에 대해서 설명을 잠깐 하자면 

Sonatype Nexus는 소프트웨어 저장소 관리 시스템으로, 주로 소프트웨어 개발에서 종속성 관리 및 아티팩트(코드, 라이브러리, 컴포넌트 등)의 저장과 관리에 사용됩니다. Nexus를 사용하면 개발자 및 개발 팀은 중앙 집중식 저장소에서 필요한 아티팩트를 검색하고, 공유하며, 관리할 수 있습니다.

우리는 보통 일반적인 개발같은 경우는 중앙 저장소 https://mvnrepository.com/ 사이트를 많이 보았을것이다 보통 아무런 설정을 하지 않으면 대부분의 개발자들은 자신도 모르게 이곳에 연결해서 라이브러리를 받게 된다 다만 이 저장소는 중앙저장소로 누구나 자신만의 라이브러리를 올릴 수 있지만 회사 자산 소스 같은 경우엔 보안에 적절하지 않으므로 
서드파티 앱으로 Nexus 를 사용하게 됩니다 그래서 이 서드파티 저장소는 중앙저장소와 분리되어서 라이브러를 관리할 수 있고 또한 개발에 필요한 라이브러리 지정된 라이브러리만 
저장하고 개발을 진행할 수 있습니다 

저는 이미 저의 VM에 2개 의 넥서스를 활용하고 있습니다 설치부터 다시 글을 써야하기에 설치할때는 임시 VM 에 설치하는 것을 보여주고 기본적인 설정이 끝이나면 제가 원래 쓰는 
Nexus 로 돌아오도록 하겠습니다 

저의 기본적인 환경은 Centos7 입니다 

## Nexus 설치 

https://help.sonatype.com/repomanager3/product-information/download 사이트로 가셔서 Unix archive 다운로드 하겠습니다 

```
wget https://download.sonatype.com/nexus/3/nexus-3.60.0-02-unix.tar.gz

```
wget 을 써서 간단하게 tar 파일을 가져 오겠습니다 rpm 이 좀 특이한게 tar 입니다 압축파을의 일종으로 알집을 풀어서 실행을 하겠습니다 

## nexus 생성 

```

useradd nexus

```

## 아카이브 옮김

```

mv ./nexus-3.60.0-02-unix.tar.gz /home/nexus/

```

## nexus 로 유저 이동하고 압축해제

```

su - nexus 

tar -zxvf ./nexus-3.60.0-02-unix.tar.gz

```

![nexus](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/24502cf7-0b55-492f-8027-76b60e2e0cf1) 

현재 페이지에 이렇게 아카이브 파일과 이를 압축푼 폴더가 생기게 됩니다 


## 실행할 User 선택

```
vi /home/nexus/nexus-3.60.0-02/bin/nexus.rc

```

입력하면 하나의 #run_as_user="" 이렇게 주석이 달려 있습니다 이 주석을 해제하고 nexus 를 입력합니다 

```
run_as_user="nexus"

```

## host , port 변경시 

```
vi /home/nexus/nexus-3.60.0-02/etc/nexus-default.properties

```

열게 되면 

```
## DO NOT EDIT - CUSTOMIZATIONS BELONG IN $data-dir/etc/nexus.properties
##
# Jetty section
application-port=8081
application-host=0.0.0.0
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml
nexus-context-path=/

# Nexus section
nexus-edition=nexus-pro-edition
nexus-features=\
 nexus-pro-feature

nexus.hazelcast.discovery.isEnabled=true
~


```

이렇식으로 포트와 host 를 지정할 수 있게 됩니다 저는 기본적인 포트와 외부 접속이 가능한 ip 인 0.0.0.0 을 사용하도록 하겠습니다 


## JDK 설치 
nexus 는 기본적으로 1.8 만 설치를 해야 진행할 수 있습니다 11 설치해서는 안되는것을 확인했습니다 

```
java-1.8.0-openjdk-devel.x86_64


```

## jdk 환경설정 

```
readlink -f /bin/java 
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.382.b05-1.el7_9.x86_64/jre/bin


vi /etc/profile 

export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.382.b05-1.el7_9.x86_64
export PATH=$PATH:$JAVA_HOME/bin


source /etc/profile
```

먼저 readlink -f /bin/java 심볼릭 링크의 원본파일을 찾아냅니다 저의 위치에서는 /usr/lib/jvm/java-11-openjdk-11.0.20.0.8-1.el7_9.x86_64/bin/java jdk 가 설치 되었습니다 다만 저희는 jre 가 필요한게 아니라 jdk 가 필요하기 때문에 JAVA_HOME 위치에는 jdk 위치를 명시해줍니다 

이를 /etc/profile 를 열고 제일 하단에 export 를 입력하하면됩니다 

source /etc/profile 이는 현재 세션에서 profile 를 적용하면됩니다 

## JDK 설치확인

```

java -version
openjdk version "1.8.0_382"
OpenJDK Runtime Environment (build 1.8.0_382-b05)
OpenJDK 64-Bit Server VM (build 25.382-b05, mixed mode)



```

이렇게 프롬프트에 입력했을때 이와 같이 나오면 설치가 잘된것입니다 

## nexus 실행

```
/home/nexus/nexus-3.60.0-02/bin/start

```

## nexus 방화벽 제거 (8081)
```
firewall-cmd --permanent --zone=public --add-port=8081/tcp

firewall-cmd --reload

```

8081 포트 방화벽 해제하고 방화벽을 재기동합니다 그러면 

![nexus 홈](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/2986e529-c7e4-405a-8847-1386f70e15a4)

이렇게 화면이 나오게 됩니다 버전업이 되면서 ui 약간 변경이 일어난거 같습니다 

## admin 로그인
```

![nexus 홈](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/11f60d50-09a3-493c-a26c-a71df1958a60)

```

로그인을 할려고 하면 다음과 같이 이 화면이 뜨면서 admin 비밀번호를 알려주게 됩니다 그리고 로그인을 하게 되면 

![nexus 홈](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f9c33b12-8151-4b6b-9ff1-067a79f173b7)

이렇게 됩니다 우리는 이렇게 nexus 의 첫발을 내딛게 되었습니다 


