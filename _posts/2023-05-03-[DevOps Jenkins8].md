---

title: DevOps Jenkins 배포 및 기동
author: kimdongy1000
date: 2023-05-03 15:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

지난시간에는 빌드 및 파라미터 활용법에 대해서 배웠습니다 오늘은 빌드된것을 가지고 배포 및 기동 작업으 진행을 해보겠습니다 사실 jenkins 라는것은 어찌되었든 리눅스 OS 안에 있는 소프트웨어로 필수 불가결하게 리눅스 명령어를 생각보다 많이 사용합니다 앞으로 jenkins - nexus 이렇게 방향을 이끌고 나갈예정입니다 이런 프로그램들은 windonw 안에서 사용할 수 있지만 현업에서 경험한 모든 프로젝트에서 윈도우 서버로 구축한 것을 본적이 없습니다 그래서 리눅스 공부도 틈틈히 공부해두시면 좋을듯 합니다 

# 새로운 User 생성 

리눅스는 모든 계정으로 움직입니다 우리가 빌드한 웹프로젝트를 실행할 계정 하나를 만들겠습니다 루트계정으로 접속하셔서 

```
useradd webAdmin 
passwd webAdmin

1234567890

```

이렇게 webAdmin 계정에 비밀번호를 입력합니다 리눅스가 비밀번호 안전성 문제를 따지지만 그냥 입력하면 넘어가게 됩니다 우리는 이 계정에 빌드한 프로젝트를 빌드 및 배포 하겠습니다 

# SSHPASS 설치 

리눅스 yum 하나 설치하겠습니다 제가 요긴하게 사용하는 것으로 ssh로 접속하여 파일을 전송 및 원격 쉘을 실행할 수 있는 라이브러리 입니다 

```

yum install -y sshpass

```

# /home/webAdmin/web 에 디렉토리 만들기 

아마 계정만 만들면 /home/webAdmin 까지 만들어지는거 까지는 확인이 될테니 그 아래에 web 이라는 폴더를 하나 더 만들어주겠습니다 
이때는 반드시 webAdmin 계정으로 접속후 만드셔야 합니다

```

su - webAdmin 

mkdir /home/webAdmin/web



```

# 파라미터 수정
자 그러면 어느정도 기초 세팅이 되었으니 다시 jenkins 로 돌아오겠습니다 지난시간까지 했던 프로젝트 여시고 몇가지 파라미터 수정 및 추가를 하겠습니다

```

REMOTEHOST = 192.168.40.132
REMOTEPASSWORD = 1234567890
REMOTEUSER = webAdmin
REMOTEPATH = /home/webAdmin/web
REMOTEPORT = 22


```

이렇게 변수명 변경 및 값을 입력해주시고 

# Execute shell 변경

```

#!/bin/bash 

echo "###############################################################"
echo "#########################배포시작###############################"

sshpass -p ${REMOTEPASSWORD} scp -o StrictHostKeyChecking=no -P ${REMOTEPORT} ${WORKSPACE}/target/*.jar ${REMOTEUSER}@${REMOTEHOST}:${REMOTEPATH}


echo "#########################배포시작###############################"
echo "###############################################################"




```

이때 조금 봐야 할것은 sshpass 의 기본적인 사용방법입니다 제가 위에서도 설명했다 싶히 이 sshpass 는 기본적으로 원격서버 파일 전송 및 원격쉘 실행입니다 

sshpass -p 원격서버 비밀번호 StrictHostKeyChecking=no 는 원격서버에 파일을 전송하거나 , 원격쉘을 실행할때 호스트의 공개키를 신뢰할 수 있는지에 대한 옵션입니다 
따라서 no 라는 옵션은 호스트의 공개키를 검사하지 않는것으로 보안에 취약할 수 있습니다 이 공개키는 지난시간에 jenkins - gitlab 에 각각 rsa_id , rsa_id_pub 에 대한것입니다 
우리는 따로 등록하지 않았음으로 no 옵션으로 가겠습니다 

-P 는 원격서버에 대한 port 옵션입니다 비밀번호는 소문자 p 입니다 

-P 포트 복사할 파일 원격서버사용자이름@원격서버host주소:파일을 복사할 위치 

sshpass -p 원격서버 비밀번호 StrictHostKeyChecking=no -P 포트 복사할 파일 원격서버사용자이름@원격서버host주소:파일을 복사할 위치 

이렇게 되는것입니다 

그리고 실행을 하게 되면 

```

###############################################################
#########################배포시작###############################
Warning: Permanently added '192.168.40.132' (ECDSA) to the list of known hosts.
#########################배포시작###############################
###############################################################
Finished: SUCCESS

```

이렇게 뜰것입니다 저 warning 는 저의 host 가 해당 주소를 알지 못하기에 known host 에 입력을 해놓겠다는 뜻입니다 

그리고 이제 우리는 webAdmin 으로 가서 /home/webAdmin/web 위치로 가면 해당 파일이 옮겨져 와 있는것을 확인할 수 있습니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/cb6a4873-547d-437e-a1ef-a411d1b2655a)

이렇게 말이죠 이때 중요한것은 단순 mv 명령어나 , cp 명령어는 해당 파일이나 디렉터리의 소유자를 변경못하지만 sshpass 는 좀 특이해서 파일을 옮길때 
jenkins 소유권한에서 우리가 사용하게 할 webAdmin 권한으로 변경까지 해줍니다 

그러면 이제 이 배포된것을 기동을 하겠습니다 이때 sshpass 사용할것데 sshpass 는 원격쉘에 파일을 복사할 수 있을 뿐만아니라 원격쉘을 복사 할 수 있습니다 

```

#!/bin/bash

echo "########################################################"
echo "서버 시작"
echo "########################################################"


SERVICE_NAME="SpringBoot-Web-Jenkins-Project"
SERVICE_PID=$(ps -ef | grep java | grep jar | grep ${SpringBoot-Web-Jenkins-Project} | awk {'print $2'})


NOHUPPATH="/usr/bin/nohup"
JAVAPATH="/usr/lib/jvm/java-11-openjdk-11.0.19.0.7-1.el7_9.x86_64/bin/java"
JARPATH=/home/webAdmin/web/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar



case ${1} in
        START)

                if [ ! -z ${SERVICE_PID} ];
                then

                        echo "############FAIL##########"
                        echo "${SERVICE_NAME} is STARTED"
                        echo "############FAIL##########"
                else
                        ${NOHUPPATH} ${JAVAPATH} -jar ${JARPATH} 1> output.log 2> error.log &
                fi
        ;;

        STOP)
                if [ ! -z ${SERVICE_PID} ];
                then
                        kill -9 ${SERVICE_PID}

                        echo "###########################"
                        echo "${SERVICE_NAME} is now stop"
                        echo "###########################"


                else
                        echo "############FAIL##########"
                        echo "${SERVICE_NAME} is STOPPED"
                        echo "############FAIL##########"
                fi



        ;;

        RESTART)
                if [ ! -z ${SERVICE_PID} ];
                then

                        echo "${SERVICE_NAME} is STARTED"

                        kill -9 ${SERVICE_PID} 

                        ${NOHUPPATH} ${JAVAPATH} -jar ${JARPATH} 1> output.log 2> error.log &
                else

                        echo "${SERVICE_NAME} is STOPPED"

                        ${NOHUPPATH} ${JAVAPATH} -jar ${JARPATH} 1> output.log 2> error.log &
                fi



        ;;
        *)
                echo "INVAILD PARAM";;
esac



```

이런 쉘을 하나 만들겠습니다 쉘 만드는것은 vi webStart.sh 이름을 이렇게 짓겠습니다 쉘 스크립트 설명은 제외하겠습니다 문법은 따로 공부하시도 
설명을 드리자면 java  switch case 로서 특정 파라미터 값이 들어오면 실행이 되는것입니다

예를 들어서 ./webStart.sh START 하면 서버가 실행되는데 이때 이미 서버가 실행중이라면 이 서버를 실행시키지 않습니다 
마찬가지로 STOP 는 서버를 멈추는데 이미 멈춰 있다면 그대로 끝이납니다 
Restart 는 멈춰 있으면 재실행하고 실행중이면 멈추고 재실행입니다 그리고 우리가 직접실행할 jar 이름이 SERVICE_NAME 으로 선언이 되어 있고 
이의 PID 를 이용해서 kill 명령어로 서버 기동을 멈추게 됩니다 

그럼 jenkins 로 돌아와서 기동 명령어를 주겠습니다 jenkins 입장에서는 restart 가 제일 효율적일것입니다 그래서 RESTART 주는 옵션으로 줘버리면

```

#!/bin/bash 

echo "###############################################################"
echo "#########################배포시작###############################"

sshpass -p ${REMOTEPASSWORD} scp -o StrictHostKeyChecking=no -P ${REMOTEPORT} ${WORKSPACE}/target/*.jar ${REMOTEUSER}@${REMOTEHOST}:${REMOTEPATH}


echo "#########################배포시작###############################"
echo "###############################################################"

sshpass -p ${REMOTEPASSWORD} ssh -o StrictHostKeyChecking=no ${REMOTEUSER}@${REMOTEHOST}  -p ${REMOTEPORT} ${REMOTESTARTSHELL} "RESTART"



```

이제는 배포하고 재기동을 우리가 본 마음대로 할 수 있게 되었습니다 그리도 기동을 하게 되면 웹에 대한 방화벽을 열어야 합니다 저는 8081 port 를 사용하니 
8081 port 는 public 하게 열겠습니다 

```

firewall-cmd --permanent --zone=public --add-port=8081/tcp
firewall-cmd --reload


```

이렇게 하고 자신의 wm 아이피 주소 + 포트로 요청을 하게 되면 

http://192.168.40.132:8081/test

아까 우리가 간단하게 만들어진 웹사이트가 보이는것을 확인할 수 있습니다