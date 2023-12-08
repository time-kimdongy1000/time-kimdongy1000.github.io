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

## tomcat 필요 없는 파일 삭제 
톰캣 설치하면 기본으로 제공하는 파일이 존재하는데 이 파일들을 삭제하겠습니다 

```
cd /home/was1/was/webapps

rm -rf ./ROOT examples docs host-manager manager

```

이 webapps 로 들어와서 저 파일들을 삭제하겠습니다 이들이 있으니 제가 배포할려는것들을 가끔 방해하는 그런것들이 생기는것을 확인해서 지웠습니다 



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

##  전체 스크립트 

```

pipeline {
    agent any
    
     environment {
        remote_tomcat_bin="/home/was1/was/bin"
        remote_tomcat_webapps="/home/was1/was/webapps"
        maven_home="/usr/share/maven/apache-maven-3.6.3"
        project_name = "SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT"
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
    
        stage('REMOTE REMOTE WEBAPP'){
            steps{
                sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} rm -rf ${remote_tomcat_webapps}/${project_name}'
                sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} rm -rf ${remote_tomcat_webapps}/${project_name}.war'
            }
        }
        
        stage('GIT PULL ORIGIN MAIN'){
            steps{
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'XXXXXXXXXXXXXX', url: 'git@gitlab.com:kimdongy1000/project_jenkins.git']])
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

이 스크립트는 

1) 원격으로 서버를 내리고

2) 원격서버에 이미 기 배포되어 있는 아카이브 삭제 

3) git 서버에서 신규 소스를 내려받고 

4) 그 소스를 WAR 패키징 진행 

5) WAR 패키징을 리모트 서버로 전송 

6) 서버 재전송

이때 2번의 로직을 좀 의야해 할 수 있는데 여러번 테스트 했을때 기 배포되어 있는 war 가 현재 배포할려는 war 에 영향을 주는거 같아서 이를 삭제했습니다 그것을 알 수 있는게 
저 스탭을 없애고 진행을 하면 신규 반영건들이 바로 적용이 되지 않고 서버를 2번 재기동해야 반영되는것을 확인 

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

이렇게 간단하긴 하지만 그래도 생각을 어느정도 해야 하는 파이프라인을 만들었습니다 그러면 된거냐 이제 부터 심화버전으로 들어갈것입니다 