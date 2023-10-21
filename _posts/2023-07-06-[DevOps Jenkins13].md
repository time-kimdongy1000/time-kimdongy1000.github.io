---

title: DevOps Jenkins PipeLine 2 파이프라인 과 Groovy 스크립트 사용
author: kimdongy1000
date: 2023-07-6 14:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 jenkins 파이프라인 프로젝트를 생성하고 간단한 파이프라인 동작과 실제 스크립트는 어떤지 살펴보았다 이번시간에는 간단한 몇가지 groovy 언어 사용을 통해서 
파이프라인 스크립트를 만들어볼것이다 

## 출력 

```

 stages {
    stage('println Hello Jenkins'){
        steps{
            println  'Hello , World!'
        }
        
    }
}

```

무엇인가 출력을 할 때는 println 을 쓰고 안에 문자열을 입력합니다 

## 변수 선언 

```

stage('declear variable'){
    steps {
        script{
            def name = "John Doe"
            println "Hello, ${name}"
        }
        
    }
}

stage('declear variable2'){
    steps {
        script{
            println "Hello, ${name}"  
        }
    }
}

```

변수는 이와 같이 steps 안에 script 를 새로 선언해서 안에 변수를 만들 수 있다 변수는 마찬가지로 {} 범위 안에서만 움직임으로 지금은 script 안에서만 유효하다 그렇기에 
stage declear variable2 에러가 발생할것이다 그래서 실행을 해보면

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/52b90983-7430-438c-a8b4-35ba48f550bf)

이렇게 한곳은 초록색 한곳은 빨간색 이렇게 보인다 이는 파이프라인도중 에러가 발생하면 이렇게 빨간색으로 표기되고 더 이상 진행지 못하고 작업을 끝내게 됩니다 

```

groovy.lang.MissingPropertyException: No such property: name for class: groovy.lang.Binding

```

로그는 이렇게 현재 정의되어 있는 name 이라는 proerty 가 무엇인지 모르겠다 입니다 즉 변수의 사용 범위를 잘 보시면됩니다 

## 전역변수 (환경변수) 
```

pipeline {
    agent any
    
     environment {
        NAME="Time"
    }
    
    stages {
        stage('println Hello Jenkins'){
            steps{
                println  'Hello , World!'
            }
            
        }
        
        stage('declear variable'){
            steps {
                script{
                  def name = "John Doe"
                  println "Hello, ${name}"
                }
                
            }
        }
        
        stage('declear variable2'){
            steps {
                script{
                    
                    def name =  env.NAME
                    println "Hello, ${name}"  
                }
            }
        }
    }
}

```

전역변수는 위치가 중요하기에 전체 파이프라인을 끌고 들어 왔다 으레 환경변수라는 명목으로 파이프라인 시작 최상단에 이렇게 environment 해두고 안에 고정 값들을 이렇게 정의할 수 있다 그럼 하단 declear variable2 에서는 name 을 evn.NAME 라는 것을 찾아서 환경변수 참조값 NAME 을 찾아서 표기를 해는 형태입니다 

```

Started by user kimdongy1000
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/Project_PipeLine_Jekins
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (println Hello Jenkins)
[Pipeline] echo
Hello , World!
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (declear variable)
[Pipeline] script
[Pipeline] {
[Pipeline] echo
Hello, John Doe
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (declear variable2)
[Pipeline] script
[Pipeline] {
[Pipeline] echo
Hello, Time
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS

```

로그를 보면 보금 난해에 보이지만 상단에 [Pipeline] withEnv 것으로 현재 이는 환경변수가 포함된 스크립트라고 생각을 하는것이고 하단에서 name 은 환경변수 evn.NAME 의 영향을 받아서 Time 이라는 전역 값을 표현을 해주게 됩니다 

## 예약값 

```

stage('where is my job location'){
    steps{
        script {
            def pwd = "${WORKSPACE}"
            println "My Job Location is ${pwd}"
            
            
        }
    }   
}




```

jenkins 에서 예약어 이미 지정되어 있는 값들은 WORKSPACE 라는 표현이 있다 이는 현재 job 에 에 대한 고정위치로 이는 변하지 않는 값이 됩니다 
이때 중요한것은 기존에 "" 안에 ${WORKSPACE} 가 들어가야 한다는것을 주의해야 합니다 이게 $를 쓸때는 항상 "" 사이에 넣어서 표기를 해주어야 합니다 

## 외부 주입값 

```

stage('This Job Parameters'){
    steps{
        script {
            def Parameter1 = "${Parameter1}"
            def Parameter2 = "${Parameter2}"
            println "Parameter1 :  ${Parameter1}"
            println "Parameter2 :  ${Parameter2}"
            
            
        }
    }   
}

```

Jenkins Job 은 단순히 파이프라인으로만 돌아가지 않는다 위에도 보았듯이 예약값이라는게 존재하고 지금할 외부 주입값이다 이 외부 주위값은 앞에서 많이 다루어보았다 
`이 빌드는 매개변수가 있습니다` 이것을 체크하면 이 Job 에서만 쓸 수 있는 외부 주입값이 생겨나게 된다 그러면 이들을 표기할때를 살펴보자 

스크립트는 간단하게 표현을 할것이다 외부주입을 하는 각각의 변수명을 Parameter1 , Parameter2 로 할것이고 우리는 위에 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/a207f164-2f9f-49a3-bb26-36fef2c602e2)

이렇게 이 빌드는 매개변수가 있습니다 클릭하면 여러가지 파라미터를 선책할 수 있는 창이 뜨는데 이게 잘 기억이 안난다면 <https://time-kimdongy1000.github.io/posts/DevOps-Jenkins7/> 를 참조하면 이에 대한 설명을 자세히 해두었다 

우리는 2개의 파라미터를 선언하고 이제 기동을 해보자 그러면 이렇게 나온다 Parameter1 는 String Paramter 로 했고 Parameter2 Password Paramter 로 진행을 했다 

```

Parameter1 :  Parameter1
[Pipeline] echo
Parameter2 :  Parameter2

```

이렇게 둘다 표기되는것을 확인할 수 있다 즉 Password Parameter 로 했을지라도 암호화나 이런것은 진행되지 않는다는것을 보여주기 위함이다 
여기서 한번 끊고 다음자에서 계속해서 groovy 스크립트에 대해서 알아볼 예정입니다 


