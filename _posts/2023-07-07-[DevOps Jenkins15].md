---

title: DevOps Jenkins 실전배포 환경 만들기 1
author: kimdongy1000
date: 2023-07-07 10:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

지난시간까지는 간단한 파이프라인 스크립트를 만들어보았다 이번시간에는 이제 그 파이프라인으로 스크립트를 작성하기 위해 환경을 먼저 조성할것이다 일단 아래 모형으로 배포 환경과 서버 환경을 구축할것이다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/dc782436-64be-4462-9d5f-739fc651cb3d)

## jdk11 설치 

이 부분은 하도 많이 해서 그렇게 어렵진 않을것이다 

```

yum install -y java-11-openjdk-devel.x86_64

```

## jdk 환경설정 

```

readlink -f /bin/java
/usr/lib/jvm/java-11-openjdk-11.0.20.0.8-1.el7_9.x86_64/bin/java

vi /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.20.0.8-1.el7_9.x86_64
export PATH=$PATH:$JAVA_HOME/bin


source /etc/profile

```

```

[root@localhost ~]# java -version
openjdk version "11.0.20" 2023-07-18 LTS
OpenJDK Runtime Environment (Red_Hat-11.0.20.0.8-1.el7_9) (build 11.0.20+8-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-11.0.20.0.8-1.el7_9) (build 11.0.20+8-LTS, mixed mode, sharing

```

이렇게 했을때 이런형식으로 뜨면 올바르게 설치가 된것입니다 

## 톰캣 설치

```

wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.82/bin/apache-tomcat-9.0.82.tar.gz

mv ./apache-tomcat-9.0.82.tar.gz /home/was1/

chown -R was1.was1 ./apache-tomcat-9.0.82.tar.gz

```

wget 으로 파일을 설치파일을 가져온뒤 이를 was1 계정으로 구동할것이기 때문에 was1 로 옮겨주고 권한까지 변경을 하겠습니다 


## tar 풀기
```

tar -zxvf ./apache-tomcat-9.0.82.tar.gz

```

tar 명령어 제일 했갈린다 중요한것은 x 가 아카이브 푸는것이고 c가 아카이브 압축이라고 생각하면된다 그러면 현재 디렉터리 (/home/was1) 안에 디렉터리가 만들어지는데 이는 우리를 이름을 변경하겠습니다 was 로 변경을 하겠습니다 

## 톰캣 기동

```

cd /home/was1/was/bin
./startup.sh

curl localhost:8080

```

했을때 html 떨어지면 올바르게 동작한것이다 

## 8080 방화벽 open


```
su - 

firewall-cmd --permanent --zone=public --add-port=8080/tcp

firewall-cmd --reload

```

결국 로컬컴퓨터에서 볼려면 8080에 대한 방화벽을 제거해야 한다 이때 방화벽은 root 권한만 해지할 수 있으니 이를 오픈을 해주고 컴퓨터 크롬으로 ip:8080 을 해보자

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/93cb055b-b91f-49f6-a668-1a8fdbf0ada0)
이렇게 나오면 톰캣 설치 완료

## 톰캣 정지 

```

cd /home/was1/was/bin
./shutdown.sh

```

shutdown.sh 를 하면 톰캣이 정지된다 자 그러면 기본적인 설치는 되었으니 이제 다음장에서 우리가 간단한 어플리케이션을 만들어서 수동으로 톰캣으로 배포한뒤 
열어보도록 하자 













