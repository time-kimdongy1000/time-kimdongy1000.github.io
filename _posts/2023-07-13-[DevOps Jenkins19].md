---
title: DevOps Jenkins 실전배포 환경 만들기 5 Credentials
author: kimdongy1000
date: 2023-07-13 10:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true
---

보안에 있어서는 정말 생명이다 특히나 서버에 접속하는 SSH 비밀번호 같은것은 노출이 되게 되면 심각한 보안사고가 일어나게 되는데 우리 앞에서 배포한것들 중에 파라미터를 한번 살펴보자 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/a38315c2-5986-4563-b55c-e156ec9021f4)

이렇게 파라미터 빌드할떄 비밀번호를 가려 놓을 수 있다 이렇게 하면 안전할꺼라고 생각하지만 파이프라인에서 보안이 쉽게 뚫리게 된다 

```

 stage('SOURCE TRANSFER REMOTE'){
    steps{
        sh 'sshpass -p ${was1_password} scp -o StrictHostKeyChecking=no -P ${was1_port} ${WORKSPACE}/target/*.war ${was1_username}@${was1_host}:${remote_tomcat_webapps}'
        
        sh 'echo was1_password : ${was1_password}'
    }
 }

```

스테이지상 비밀번호 파라미터를 쓰는곳에 가서 `sh 'echo was1_password : ${was1_password}'` 이렇게 입력을 하게 되면 콘솔에 비밀번호가 그대로 노출이 됩니다 
물론 jenkins 라는것은 아무나 들어올 수 없는 공간이긴 하지만 jenkins admin 권한의 비밀번호가 털리게 되면 자동으로 서버의 정보 + ssh 비밀번호가 까지 날라가게 된다 
그래서 jenkins 는 Credentials 라고 한가지를 제공하는데 이곳에서 완벽한 비밀번호를 가릴 수가 있게 됩니다 

## Credentials

Dashboard > Jenkins 관리 > Credentials

들어오시면 Stores scoped to Jenkins 가 있는데 이곳을 클릭해서 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/92523da0-8d6e-4589-adf3-e35c5fd85d43)

하단에 Add Credentials 를 클릭합니다

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/448b3c87-3f25-4836-a2c5-8ee903a7d89a)

그리고 사진과 같이 작성을 합니다 이때 scret key 가 ssh 비밀번호가 될 예정입니다 Save 해서 나오게 되면 이제 이를 파이프라인에 적용을 하겠습니다


## 파이프라인에 Credentials 적용
기존의 Password Parameter 는 빌드 파라미터에서 삭제를 하겠습니다 그리고 이 비밀번호가 필요한 모든 공간에서 이와 같이 작성을 합니다 


```

script{
    withCredentials([string(credentialsId: 'WAS1_SSH_PASSWORD', variable: 'PASSWORD')]) {
        sh 'sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/shutdown.sh'     
    }

}

```

이렇게 withCredentials 이라는 것을 쓰고 credentialsId 는 위의 사진에서 ID 를 입력합니다 variable 해당 노드에서 쓸 비밀번호 변수를 정의합니다 그러면 끝납니다 
전체 스크립트는 이렇게 바뀔것입니다

```

pipeline {
    agent any
    
     environment {
        remote_tomcat_bin="/home/was1/was/bin"
        remote_tomcat_webapps="/home/was1/was/webapps"
        maven_home="/usr/share/maven/apache-maven-3.6.3"
        project_name = "SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT"
        
        remote_address = "http://192.168.46.132:8080"
        remote_helthCheck_handler = "http://192.168.46.132:8080/health"
     
        
      
        
    }
    
    
    stages {
        stage('START_JENKINS_PIPLLINE'){
            steps{
                println  'START DEPLOY WEBAPP'
            }
        }
        
      stage('STAUS REMOTE SERVER'){
            steps {
                script{
                    
                    def health_response_text = "";
                
                    for(int i = 0; i < 3; i++){
                        health_response_text = sh(script: ' curl -s http://192.168.46.132:8080/health' , returnStatus: true)
                        
                        if(health_response_text != 0){
                            sleep 1
                             
                        }else{
                            break;
                        }
                    }
                    
                    env.health_response_code = health_response_text
                     println  "${env.health_response_code}"
                    

                }
                
            }
        }
        
         stage('STOP REMOTE TOMECAT'){
             when {
                 expression  {health_response_code == "0"} 
             }
            steps {
                script{
                     withCredentials([string(credentialsId: 'WAS1_SSH_PASSWORD', variable: 'PASSWORD')]) {
                         
                        sh 'sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/shutdown.sh'     
                    }
                    
                }
                
            }
            
        }
    
        stage('REMOTE REMOTE WEBAPP'){
            steps{
                
                script{
                     withCredentials([string(credentialsId: 'WAS1_SSH_PASSWORD', variable: 'PASSWORD')]) {
                        
                        sh 'sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} rm -rf ${remote_tomcat_webapps}/${project_name}'
                        sh 'sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} rm -rf ${remote_tomcat_webapps}/${project_name}.war'     
                    }
                    
                }
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
                script{
                    withCredentials([string(credentialsId: 'WAS1_SSH_PASSWORD', variable: 'PASSWORD')]) {
                        sh 'sshpass -p ${PASSWORD} scp -o StrictHostKeyChecking=no -P ${was1_port} ${WORKSPACE}/target/*.war ${was1_username}@${was1_host}:${remote_tomcat_webapps}'    
                    }
                    
                }
                
            }
        }
        
         stage('START REMOTE TOMECAT'){
             steps{
                 script{
                    withCredentials([string(credentialsId: 'WAS1_SSH_PASSWORD', variable: 'PASSWORD')]) {
                        
                    sh 'sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/startup.sh'            
                    }
                }
             }
        }
        
    }
}

```

이렇게 바뀌게 되고 그럼 똑같이 생각을 할수 있는데 마찬가지로 echo 를 써서 비밀번호를 한번 탈취를 해보자 

```

stage('START REMOTE TOMECAT'){
        steps{
            script{
            withCredentials([string(credentialsId: 'WAS1_SSH_PASSWORD', variable: 'PASSWORD')]) {
                
            sh 'sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/startup.sh'            
            sh 'echo PASSWORD :  ${PASSWORD}'
            }
        }
    }
}

```

이렇게 하면 비밀번호는 어떻게 나오게 될까?

```

+ sshpass -p **** ssh -o StrictHostKeyChecking=no was1@192.168.46.132 -p 22 /home/was1/was/bin/startup.sh
Tomcat started.
[Pipeline] sh
+ echo PASSWORD : ****
PASSWORD : ****

```

모든 sshpass 는 이렇게 비밀번호가 *** 로 표기됩니다 이렇게 하면 설사 jenkins 관리자를 탈취당해도 ssh 비밀번호까지는 지킬 수 있습니다 
