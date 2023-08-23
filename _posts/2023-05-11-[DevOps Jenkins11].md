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

이번장에는 대대적인 수정작업이 있을 예정입니다 


## 배포 스크립트 수정 

```

#!/bin/bash 


echo "GIT_BRANCH :  ${GIT_BRANCH}"


echo "###############################################################"
echo "#########################배포시작###############################"

if [ ${GIT_BRANCH} == "origin/dev" ];
then 
	sshpass -p ${REMOTEPASSWORD} scp -o StrictHostKeyChecking=no -P ${REMOTEPORT} ${WORKSPACE}/target/*.jar ${REMOTEUSER}@${REMOTEHOST}:${REMOTEPATH}/dev
    
elif [ ${GIT_BRANCH} == "origin/main" ];
then
	sshpass -p ${REMOTEPASSWORD} scp -o StrictHostKeyChecking=no -P ${REMOTEPORT} ${WORKSPACE}/target/*.jar ${REMOTEUSER}@${REMOTEHOST}:${REMOTEPATH}/prod
fi    
	


if [ ${GIT_BRANCH} == "origin/dev" ];
then 
	sshpass -p ${REMOTEPASSWORD} ssh -o StrictHostKeyChecking=no ${REMOTEUSER}@${REMOTEHOST}  -p ${REMOTEPORT} ${REMOTESTARTSHELL} "RESTART" "DEV"
    
elif [ ${GIT_BRANCH} == "origin/main" ];
then
	sshpass -p ${REMOTEPASSWORD} ssh -o StrictHostKeyChecking=no ${REMOTEUSER}@${REMOTEHOST}  -p ${REMOTEPORT} ${REMOTESTARTSHELL} "RESTART" "PROD"
fi 

echo "#########################배포시작###############################"
echo "###############################################################"




```

기존 배포 스크립트가 변경될 예정입니다 이때 jenkins 예약어로 GIT_BRANCH 를 사용할 수 있는데 현재 분기 되어서 pull 받아서 진행을 하고 있는 브랜치 명을 알려주게 됩니다 
이 브랜치 명을 기준으로 위와 같은 배포 스크립트가 만들어지게 됩니다 

첫번쨰 분기는 브랜치에 따라서 dev 에 배포할것인지 prod 에 배포 할 것인지 위치를 먼저 정하고 파일을 복사해서 던집니다 
두번째 분기는 브랜치에 따라서 어떤 모드로 기동할것인지 인자를 주는것입니다 

## 기동 스크립트 수정 

```

#!/bin/bash

echo "########################################################"
echo "서버 시작"
echo "########################################################"


SERVICE_NAME="SpringBoot-Web-Jenkins-Project"
#SERVICE_PID=$(ps -ef | grep java | grep jar | grep SpringBoot-Web-Jenkins-Project | awk {'print $2'})

## 각 실행위치에 따라서 pid 를 각각 가져오기 
SERVICE_DEV_PID=$(ps -ef | grep java | grep jar | grep /home/webAdmin/web/dev/SpringBoot-Web-Jenkins-Project | awk {'print $2'})
SERVICE_PRD_PID=$(ps -ef | grep java | grep jar | grep /home/webAdmin/web/prod/SpringBoot-Web-Jenkins-Project | awk {'print $2'})


NOHUPPATH="/usr/bin/nohup"
JAVAPATH="/usr/lib/jvm/java-11-openjdk-11.0.18.0.10-1.el7_9.x86_64/bin/java"

#JARPATH=/home/webAdmin/web/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar

## 개발/운영 jar 파일 위치 명시
JAVADEVPATH=/home/webAdmin/web/dev/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar
JAVAPRODPATH=/home/webAdmin/web/prod/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar

## 개발/운영 STANDARD 로그 명시
JAVADEVLOG=/home/webAdmin/web/dev/DEV.log
JAVADEVERRORLOG=/home/webAdmin/web/dev/ERROR.log

## 개발/운영 ERROR 로그 명시
JAVAPRODLOG=/home/webAdmin/web/prod/PROD.log
JAVAPRODERRORLOG=/home/webAdmin/web/prod/ERROR.log


## 분기에 따른 서버 기동 함수 
WEBSERVER_TIGGER(){

        echo ${1}

        if [ "DEV" == ${1} ];
        then
                ${NOHUPPATH} ${JAVAPATH} -jar ${JAVADEVPATH} 1> ${JAVADEVLOG} 2> ${JAVADEVERRORLOG} &

        elif [ "PROD" == ${1} ];
        then
                ${NOHUPPATH} ${JAVAPATH} -jar ${JAVAPRODPATH} 1> ${JAVAPRODLOG} 2> ${JAVAPRODERRORLOG} &
        fi

}

## 분기에 따른 서버 stop 명령 
KILL_WEBSERIVE(){

        echo ${1}

        if [ "DEV" == ${1} ];
        then

                kill -9 ${SERVICE_DEV_PID}

        elif [ "PROD" == ${1} ];
        then
                kill -9 ${SERVICE_PRD_PID}
        fi

}


## 분기에 따른 현재 기동중인 서버에 대한 PID 반환 
RETURN_WEBPID=""
RETURN_WEBPID(){

        echo "RETURN_WEBPID" ${1}

        if [ "DEV" == ${1} ];
        then

                RETURN_WEBPID=${SERVICE_DEV_PID}
                echo ${RETURN_WEBPID}


        elif [ "PROD" == ${1} ];
        then
                RETURN_WEBPID=${SERVICE_PRD_PID}
                echo ${RETURN_WEBPID}
        fi

}

RETURN_WEBPID ${2}



case ${1} in
        START)

                if [ ! -z "$RETURN_WEBPID" ];
                then

                        echo "############FAIL##########"

                        if [ "DEV" == ${2} ];
                        then
                                echo "DEV_${SERVICE_NAME} is STARTED"

                        elif [ "PROD" == ${2} ];
                        then
                                echo "PROD_${SERVICE_NAME} is STARTED"
                        fi


                        echo "############FAIL##########"

                else
                        WEBSERVER_TIGGER ${2}
                fi
        ;;

        STOP)
                if [ ! -z "$RETURN_WEBPID" ];
                then
                        KILL_WEBSERIVE ${2}

                        echo "###########################"

                        if [ "DEV" == ${2} ];
                        then
                                echo "DEV_${SERVICE_NAME} is STOPPED"

                        elif [ "PROD" == ${2} ];
                        then
                                echo "PROD_${SERVICE_NAME} is STOPPED"
                        fi


                        echo "###########################"


                else
                        echo "############FAIL##########"

                        if [ "DEV" == ${2} ];
                        then
                                echo "DEV_${SERVICE_NAME} was STOPPED"

                        elif [ "PROD" == ${2} ];
                        then
                                echo "PROD_${SERVICE_NAME} was STOPPED"
                        fi


                        echo "############FAIL##########"
                fi



        ;;

        RESTART)

                if [ ! -z "$RETURN_WEBPID" ];
                then

                        if [ "DEV" == ${2} ];
                        then
                                echo "DEV_${SERVICE_NAME} was RESTARTED1"

                        elif [ "PROD" == ${2} ];
                        then
                                echo "PROD_${SERVICE_NAME} was RESTARTED1"
                        fi

                        KILL_WEBSERIVE ${2}

                        WEBSERVER_TIGGER ${2}


                else
                        if [ "DEV" == ${2} ];
                        then
                                echo "DEV_${SERVICE_NAME} was RESTARTED2"

                        elif [ "PROD" == ${2} ];
                        then
                                echo "PROD_${SERVICE_NAME} was RESTARTED2"
                        fi

                        WEBSERVER_TIGGER ${2}

                fi



        ;;

        *)
                echo "INVAILD PARAM";;
esac


```
지금 보면 지난 시간까지 너무나도 스크립트를 보여주고 있습니다 기존에는 간단하게 만들었지만 내용도 커지고 안에 고려해여할 대상이 많아지다 보니 
기존 스크립트로는 도저히 안되어서 전부 함수화 시키고 바깥으로 뺴었습니다 간단한 주석이 달려있으니 읽어보시면 충분히 이해하실 수 있습니다 

그럼 이제 로컬 개발자가 각 DEV 로 배포를 진행하게 되면 배포스크립트에 따라서 배포가 진행이 되고 기동 스크립트에 따라서 기동이 되게 됩니다 
각 분기에 따라서 dev , prod 로 들어가서 각자의 역활에 맞게 배포 및 기동이 한번에 되는 스크립트까지 만들어보았습니다 


복습하면 우리는 지난시간까지 간단한 웹훅 연결부터 이번 포스트의 웹훅에 따른 적절한 위치 배포 및 기동까지 다루어 보았습니다 
우리는 이것을 더욱 발전을 시켜서 무중단 배포까지 해볼 예정입니다.

