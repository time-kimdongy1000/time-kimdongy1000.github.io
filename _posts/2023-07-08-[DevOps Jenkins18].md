---

title: DevOps Jenkins 실전배포 환경 만들기 4
author: kimdongy1000
date: 2023-07-08 14:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

지난시간에 전체적인 스크립트를 작성을 했다 저것만 있으면 배포를 할 수는 있다 다만 파이프라인 스크립트를 쓰는것은 여러 상황에 대한 대처와 에러 발생시 어떻게 할것인지에 대한 
다양한 상황이 존재하며 그 상황이 발생할때마다 그에 필요한 스크립트를 작성을 해주었다 우리는 내가 실무에서 보면서 이런 상황이 발생했을때 이런 스크립트가 있었으면 좋았을거 같은 몇가지를 적으면서 진행을 하겠습니다 

## 상황 1
상황 1 은 배포할려는 서버가 꺼져 있는 상황이다 사실 서버가 꺼져 있어도 스크립트 자체는 오류는 아래 처럼 발생하지만 파이프라인 자체는 흘러가게 되고 
최종적으로는 실패가 떨어지게 된다 

```

심각: [localhost:8005] (base 포트 [8005] 그리고 offset [0])와(과) 연결할 수 없었습니다. Tomcat이 실행 중이지 않을 수 있습니다.
10월 22, 2023 7:01:29 오후 org.apache.catalina.startup.Catalina stopServer
심각: Catalina를 중지시키는 중 오류 발생
java.net.ConnectException: 연결이 거부됨 (Connection refused)
	at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
	at java.base/java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:412)
	at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:255)
	at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:237)
	at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
	at java.base/java.net.Socket.connect(Socket.java:609)
	at java.base/java.net.Socket.connect(Socket.java:558)
	at java.base/java.net.Socket.<init>(Socket.java:454)
	at java.base/java.net.Socket.<init>(Socket.java:231)
	at org.apache.catalina.startup.Catalina.stopServer(Catalina.java:667)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
	at org.apache.catalina.startup.Bootstrap.stopServer(Bootstrap.java:393)
	at org.apache.catalina.startup.Bootstrap.main(Bootstrap.java:483)

```

다만 jenkins 입장에서는 실패가 맞지만 원격 서버 입장에서는 배포가 되어서 서버가 재기동 된거처럼 보인다 이런 상황을 해결하기 위해선 2가 방법이 보이는데 

1) 서버가 내려가 있는 상태에서는 배포 할 수 없는 방식 

2) 서버가 내려가 있으면 서버는 내리는 파이프 라인을 태우지 않는 방법이다 굳이 실무가 아니라도 당연히 1번보다는 2번이 jenkins 관리적인 측면이나 유연하기 때문에 2번을 많이 쓴다 2번을 쓰기 위해서는 현재 remote 서버의 상태를 알 수 있어야 한다 이런것을 보통 helthCheck 라고 하는데 요고를 한번 만들어보겠다


## Helthcheck 로직 추가 

```

package com.cybb.main.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("health")
public class HealthCheckController {
    
    @GetMapping("")
    public ResponseEntity<?> responseHealth(){
        
        return new ResponseEntity<>( , HttpStatus.OK);
    }
}


```

이러한 로직은 사용하는 환경에 따라서 현재 서버 상태를 반환하는 무엇인가 있을 수도 있고 없을 수도 있다 그런것들을 담당자가 잘 파악해서 활용하는것이 좋다 
예를 들어서 지금 이 프로젝트는 이런 핸들러를 하나 추가할것이다 여기서 1번과 HttpStatus 200 을 호출하면 이는 현재 서버가 올라가 있는 상태이다 


## 파이프란 구축 (서버가 이미 내려갔을 상황)

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

설명을 드리면 

```

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

```
이 부분에서는 우리가 만든 health 핸들러와 통신을 시도한다 이때 서버가 켜져 있으면 health_response_text 는 0을 반환하고 그렇지 않으면 7을 반환다 
이때7은 고정값으로 서버와 연결을 할 수 없다는것을 알리는 코드이다 다만 이는 한번 try 해서 바로 판단하지 않고 여러번 반복문을 돌려서 서버의 상태를 체크한다 
이때 sleep 는 편의상 1을 줬다 나는 다음부터는 10을 주고 총 30초 동안 서버의 상태를 파악할 것이다 

그리고 그 변수값을 환경변수에 저장을 해두고 전역 변수로 활용을 할것이다 

```

 stage('STOP REMOTE TOMECAT'){
		when {
			expression  {health_response_code == "0"} 
		}
	steps {
		sh 'sshpass -p ${was1_password} ssh -o StrictHostKeyChecking=no ${was1_username}@${was1_host}  -p ${was1_port} ${remote_tomcat_bin}/shutdown.sh'
	}
	
}

```

이 스테이지를 살펴보자 wehn 을 파이프라인에서 if 문 같은것이다 이때 when 을 만족하면 이 stage 르 실행고 그렇지 않으면 건너뛰게 된다 
즉 위에서 환경변수가 0번을 저장하고 있으면 이 스테이지를 실행을 하고 그렇지 않으면 뛰어넘게 된다 

그렴 파이프란 보기는 이와 같게 된다 

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/92f9883b-25aa-4ca4-ab2f-6cbe5044a6ac)

85번 실행결과를 보자 STOP REMOTE TOMCAT 이 비어 있는것을 보이는가 즉 이때는 서버가 꺼져 있기 때문에 서버를 다시 끌 필요 없이 다음 stage 로 넘어가는 상황이다 


이때 로그를 살펴보게 되면 이런 모양으로 나오게 된다 

```

Sleeping for 1 sec
[Pipeline] echo
7
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (STOP REMOTE TOMECAT)
Stage "STOP REMOTE TOMECAT" skipped due to when conditional

```

Stage "STOP REMOTE TOMECAT" skipped due to when conditional 이렇게 이 stage 를 띄어 넘겠다는 뜻이다 그리고 83번 실행을 살펴보면 when 을 만족하기 때문에 
그대로 실행을 하는 모습을 보이고 있는것이다 자 우리는 첫번째 상황에 대한 파이프라인 대처를 만들었다 

앞으로 파이프라인은 내가 실무에서 겪은 또는 내가 앞으로 구축하고 싶은 상황을 부여해서 스크립트를 만들어나갈 계획입니다 <2023-07-08-[DevOps Jenkins18].md>