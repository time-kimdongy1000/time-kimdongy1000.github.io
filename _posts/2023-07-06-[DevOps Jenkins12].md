---

title: DevOps Jenkins PipeLine
author: kimdongy1000
date: 2023-07-6 10:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

지난시간에 우리는 jenkins webhook 을 진행하고 꽤나 오랫동안 jenkins 관련한 글을 적지 않았다 당분간 시큐리티 글이 없을 예정이기에 다음 시큐리티 까지는 jenkins 파이프라인에 대해서 알아볼려고 한다 지금부터 적는 모든 내용은 jenkins 의 정석은 아니다 오랫동안 배포 담당자로 일을 하고 있었지만 파이프라인으로 구축해본 프로젝트는 한번도 없었다 그렇기에 내가 하고 싶은거 그리고 다음프로젝트 가서 적용을 해볼 수 있을만한 내용으로 정리를 할 예정이다 

## Jenkins PipeLine 이란 
파이프라인은 우리가 잘 알고 있다 싶히 수도관을 생각하면 된다 우리는 그 수도관을 구축하는 임무를 맡을 예정이며 그 수도관에 흐르는 것들은 배포가 되는 소스들이 
흘러갈 예정이다 그럼 파이프라인이 무엇인지 정의부터 알아보자 

Jenkins 파이프라인은 지속적 통합 및 지속적 전달 (CI/CD) 프로세스를 자동화하고 관리하는 데 사용되는 Jenkins의 확장 기능 중 하나입니다. 파이프라인은 소프트웨어 개발 및 배포 프로세스를 스크립트로 정의하고 관리할 수 있는 강력한 도구입니다.

즉 우리는 앞에서 쉘스크립트 작성을 통해서 파이프라인을 구축했다 이때의 언어는 쉘스크립트 였다면 jenkins 가 자체로 지원하는 파이프라인 언어는 Groovy 언어이다 
java 기반으로 출발한 언어이기 때문에 그렇게 어렵게 느껴지지는 않을것이다 

## PipeLine 구축 

먼저 파이프라인 프로젝트를 만들어야 한다 좌측에 새로운 Item 클릭하고 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/972cea42-2e18-4eda-97b7-d6c7a9c6d64c)

이렇게 이름을 짓고 PipeLine 를 클릭을 하면된다 그러면 위에 설정은 뒤로 물리고 제일 하단에 Pipeline 스크립트를 쓸 수 있는 공간이 있다 
하단에 PipeLine syntax 를 클릭하면 예제 몇개가 주어지긴 하지만 일단 우리는 이와 같이 적어볼것이다 

```

pipeline {
    agent any
    
    stages {
        stage('println Hello World') {
            steps {
                println  'Hello World!'
            }
        }
    }
}

```

이와 같이 적을 것이다 그리고 저장을 하고 나와서 지금 빌드를 클릭하면 이와 같은 화면이 나올것이다 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/a069172a-2796-474d-832d-720ec3003820)

이렇게 말이다 

그리고 log 를 보면 

```

Started by user kimdongy1000
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/Project_PipeLine_Jekins
[Pipeline] {
[Pipeline] stage
[Pipeline] { (println Hello World)
[Pipeline] echo
Hello World!
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS

```

이와같이 이런 모습으로 보이고 끝이나게 된다 그러면 하나씩 설명을 해보자 

1) pipeline {} 이 부분은 최초 파이프라인 선언 부분이다 이 이 아래 {} 부터 파이프라인으로 인식을 한다는것이고 여기서 부터 시작을 하겠다는 뜻이다 

2) agent any 이 자체가 가지는 의미는 jenkins 파이프라인을 어떤 에이전트에서 실행할 수 있도록 허용을 한다는 뜻입니다 

3) stages 이는 stage 의 모음으로 생각하면 되고 파이프라인의 논리적인 구성들을 모아놓은 스크립트 집합입니다 
   이 부분에서 우리는 해야 할 일을 논리적으로 정의하고 에러를 처리하는 부분이 바로 이 부분입니다 

4) stage 위에서 보았듯이 stage 여러개가 모여서 stages 가 됨으로 이 stage 는 작업의 하나의 단계입니다 

5) steps 실제 stage 가 정의한 내용을 코드로 옮겨적는 부분입니다 

제가 stage 에 println Hello World 적어 놓았으니 이 stage 에서는 Hello World 프린트 하는것이 목적임으로 echo Hello World~ 이렇게 코드로 옮겨 적어 놓은것입니다 
즉 모든 그루비 스크립트는 여기에서 행해질 예정입니다 

## 예제 그루비 스크립트 
다음 장부터는 그루비 언어를 어떻게 하면 효과적으로 사용할 수 있는지 해서 정리할 예정이고 그 전에 jenkins 가 제공하는 pipeLine syntax 에 대해서 알아보도록 하자


![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/d74fde2a-208b-49c2-973f-9d0af03c904f)

여기에 우리는 git 이라고 해두고 아래와 같이 우리가 앞에서 만든 프로젝트를 사용을 해보자 ssh 인증이 기억이 나지 않는다면 
<https://time-kimdongy1000.github.io/posts/DevOps-Jenkins5/> 다시 한번 쭈욱 읽어보면 private 한 프로젝트 소스를 어떻게 가져오는지 정리를 해두었다 


## 스크립트 작성 

```

pipeline {
    agent any
    
    stages {
        stage('println Hello World') {
            steps {
                println  'Hello , World!'
            }
        }
        
        stage('git pull origin/man'){
            steps {
                git credentialsId: 'xxxxxxxxxxxx', url: 'git@gitlab.com:kimdongy1000/project_jenkins.git'
            }
        }
    }
}

```
돌아와서 pipeLine 를 이와 같이 작성을 한다 credentialsId 값은 모자이크 처리한거니 양해를 바랍니다 그럼 실행을 하게 되면 

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ecbce4b1-e568-4dba-92da-26bd52828e5e)

메인 화면에 하나의 스테이지가 늘어난것을 볼 수 있고 로그를 들어가서보면 

```

Started by user kimdongy1000
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/Project_PipeLine_Jekins
[Pipeline] {
[Pipeline] stage
[Pipeline] { (println Hello World)
[Pipeline] echo
Hello , World!
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (git pull origin/man)
[Pipeline] git
The recommended git tool is: NONE
using credential xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Cloning the remote Git repository
Cloning repository git@gitlab.com:kimdongy1000/project_jenkins.git
 > git init /var/lib/jenkins/workspace/Project_PipeLine_Jekins # timeout=10
Fetching upstream changes from git@gitlab.com:kimdongy1000/project_jenkins.git
 > git --version # timeout=10
 > git --version # 'git version 1.8.3.1'
using GIT_SSH to set credentials PROD1
[INFO] Currently running in a labeled security context
[INFO] Currently SELinux is 'enforcing' on the host
 > /usr/bin/chcon --type=ssh_home_t /var/lib/jenkins/workspace/Project_PipeLine_Jekins@tmp/jenkins-gitclient-ssh2293020989539746141.key
Verifying host key using known hosts file
 > git fetch --tags --progress git@gitlab.com:kimdongy1000/project_jenkins.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git config remote.origin.url git@gitlab.com:kimdongy1000/project_jenkins.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
Avoid second fetch
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
Checking out Revision 5ab84c87b8daf8c045fd39f5d2dc2c80ecf824f6 (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 5ab84c87b8daf8c045fd39f5d2dc2c80ecf824f6 # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git checkout -b master 5ab84c87b8daf8c045fd39f5d2dc2c80ecf824f6 # timeout=10
Commit message: "서버 포트 변경"
First time build. Skipping changelog.
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS

```

이렇게 소스를 새롭게 받아오는 모습을 볼 수 있다 물론 소스가 받아지는 위치는 /var/lib/jenkins/workspace/Project_PipeLine_Jekins 위치가 될것이다 
우리가 파이프라인을 쓴다고 해도 기본적인 파일의 위치는 변함이 없게 된다 여기에 들어오면 이제 우리가 방금받은 소스가 들어가 있는 것을 확인할 수 있다 

우리는 이번시간에 jenkins 파이프라인에 대해서 공부 해보았고 다음장부터는 groovy 언어를 조금만 공부해서 내가 구축하고 싶은 파이프라인을 하나씩 만을어 갈것이다 