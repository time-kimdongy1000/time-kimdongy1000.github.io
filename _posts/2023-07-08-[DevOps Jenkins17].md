---

title: DevOps Jenkins 실전배포 환경 만들기 3
author: kimdongy1000
date: 2023-07-08 10:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 프로젝트를 하나 만들어서 직접 손으로 배포하는 과정을 만들어 보았습니다 손으로 하니 이걸 일일이 할 수도 있지 않은가 할 수 있습니다 
지금 우리가 보는것은 프로젝트 단 하나 배포 하는것이며 실전에는 약 200개의 프로젝트를 톰캣 안에 집어놓고 기동을 하게 됩니다 그때마다 수동으로 war , jar 를 말고 보낼 수는 없기에 우리는 jenkins 를 통해서 좀더 쉽게 지속적 배포를 할 수 있게 만들어보겠습니다 



## 파이프라인 스크립트 
처음엔 간단한 파이프라인으로 구축해서 우리가 앞에서 했던것처럼 배포가 되고 서버가 내려갔다 올라갔다 정도를 구현을 하고 여기서 점점 많은 기능을 붙여서 정말 다양한 기능으로 
지속적 배포 스크립트를 만들어보겠습니다 

새로운 파이프라인 프로젝트를 만들것입니다 이름은 webapp_piple 으로 이름을 짓겠습니다 

이 빌드는 매개변수가 존재합니다 

1번 서버와 -> was1 번으로 전송을 하는것임으로 was1 에 대한 정보를 기입하겠습니다 

내가 설정할 매개변수는 아래와 같습니다 각자 서버 상황이 다르니 값에 대한 구체적인 언급은 하지 않겠습니다

was1_host
was1 의 아이피 주소

was1_password
was1 의 password 

was1_username
was1 의 username

was1_port
was1 의 port 

로직은 앞에서 했던것과 동일하다 소스 내려받고 sshpass 쏘고 이상 없으면 기동하고 이런 형식이 일단 간단한 기동 절차가 되는것이다 물론 이내용은 pipleLine 의 환경변수로 남겨두어도 좋습니다 

## 서버 내리기 올리기 

```

pipeline {
    agent any
    
     environment {
        remote_tomcat_bin="/home/was1/was/bin"
        remote_tomcat_webapps="/home/was1/was/webapps"
    }
    
    
    stages {
        stage('START_JENKINS_PIPLLINE'){
            steps{
                println  'START DEPLOY WEBAPP'
            }
        }
        
        stage('STOP REMOTE TOMECAT'){
            steps {
                sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/shutdown.sh'
            }
            
        }
         
        stage('START REMOTE TOMECAT'){
            steps{
              sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/startup.sh'  
            }
        }        
    }
}

```

일단 원격으로 서버를 내렸다가 올리는거 먼저 해보자 이렇게 구성을 하고 진행을 하면 원격으로 서버가 내려고 원격으로 올라가는 모습을 볼 수 있다 우리는 이 사이에 
소스를 내려 받고 그것을 wepapp path 에 옮겨놓아보자 그리고 소스 수정여부를 확인하기 위해 간단한 핸들러 하나를 더 추가해서 올릴것이다 참고로 앞에서 우리 환경변수 참조할때 env 썻는데 그냥 $ 표기해도 쓸 수 있다 

## 간단한 핸들러추가

```

package com.cybb.main.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("demo")
public class DemoController {

    @GetMapping("")
    public String demoController(){

        return "demoController";
    }

    @GetMapping("/2")
    public String demoController2(){

        return "demoController2";
    }
}


```

간단하게 2 라는 핸들러 하나 추가해서 올리고 이게 올바르게 반영되었으면 화면에 demoController2 나오는것을 알 수 있을것이다 이 내용을 반영을 하고 올리겠다 

## 나머지 스크립트 내용 추가 

```

pipeline {
    agent any
    
     environment {
        remote_tomcat_bin="/home/was1/was/bin"
        remote_tomcat_webapps="/home/was1/was/webapps"
        maven_home="/usr/share/maven/apache-maven-3.6.3"
    }
    
    
    stages {
        stage('START_JENKINS_PIPLLINE'){
            steps{
                println  'START DEPLOY WEBAPP'
            }
        }
        
        stage('STOP REMOTE TOMECAT'){
            steps {
                sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/shutdown.sh'
            }
            
        }
        
        stage('GIT PULL ORIGIN MAIN'){
            steps{
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'f0a15a3e-e71d-49c4-89f4-43856f10380e', url: 'git@gitlab.com:kimdongy1000/project_jenkins.git']])
            }
        }
        
        stage('BUild AND WAR Packing'){
            steps{ 
                sh '${maven_home}/bin/mvn -f  ${WORKSPACE} clean package' 
            }
        }
        
        stage('SOURCE TRANSFER REMOTE'){
            steps{
                sh 'sshpass -p ${was1_password} scp -o StrictHostKeyChecking=no -P ${was1_port} ${WORKSPACE}/target/*.war ${was1_username}@${was1_host}:${remote_tomcat_webapps}'
            }
        }
        
        
        
        stage('START REMOTE TOMECAT'){
            steps{
              sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/startup.sh'  
            }
        }        
    }
}

```
이제 나머지 핵심적인 내용은 소스를 내려 받고 그 소스를 WAR 패키징을 한 뒤에 그것을 원격서버로 전송을 한뒤에 서버를 올리는거 지극히 앞의 1편하고 다를게 없어보인다 그럼 왜 하는지에 대해서는 마지막에 기술하기로 하고 올바르게 전소이 되어서 올라갔는지 확인을 해보자

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/955458c5-cef9-4986-ba59-d47f2acd819b)

그럼 이렇게 바깥에 표기될것이다 일단 이는 파이프라인으로 구축한것이기 때문이지만 결국 하는 행동은 단일 스크립트이다 하나의 경우가 실패하면 뒤 내용이 실행되지 않는다 그럼에서 파이프라인으로 구축할려는 이유는 결국은 파이프라인을 병렬적으로 구축해서 다양한 방식으로 구축과 에러 컨트롤을 통해서 여러가지 방식으로 구축할 계획이다 일단은 그런 어려운 방향으로 나아가기 위해서는 단일 스크립트로 배포를 했을 시 올바르게 잘 나가는지 먼저 파악하는게 우선이다 

잠깐 다른소리로 들아와서 계속해서 신규로 적는 핸들러가 바로 적용이 안되는 현상을 발견했다 이상하게 서버를 2번 껏다 켜야 적용이 되는것을 보았는데 이는 server.xml 을 수정하면 자동으로 반영이 된다

## server.xml 수정

```
<Context docBase="SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT" path="/" reloadable="true"/>

```

저기에서 reloadable 앞에서는 false 로 되어 있을텐데 이를 true 로 옮기면 된다 그러면 잠시뒤 톰캣이 변경점을 감지해서 이를 적용하게 됩니다 오늘은 여기 까지 정리하겠습니다 







