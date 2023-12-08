---

title: DevOps Jenkins PipeLine 3 파이프라인 과 Groovy 스크립트 사용
author: kimdongy1000
date: 2023-07-06 16:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 파이프라인 여러가지 사용법가 groovy 스크립트에 대해서 공부를 했습니다 계속해서 이어나가겠습니다 

## 반복문 

```

stage('loop stage'){
    steps {
        script {
                for (int i = 1; i <= 5; i++) {
                println i
            }
        }
    }
}

```

당연한것이겠지만 반복문을 사용할 수 있다 1 에서 5까지 반복을 하는것이고 이때 중요한것은 단일 스크립트 상태이기 때문에 이 반복문이 끝나기 전에는 다음 stage 로 넘어가지 못합니다 우리는 뒤에 병렬 스크립트를 배울때 이 for 문이 돌때 다른 작업을 돌려서 진행을 하게끔 설계를 할 것입니다 

## 반복문 2

```

 stage('declear variable loop Num'){
    steps {
        script {
            
            int loop_num = "${loo_num}".toInteger();
            for(int i = 1; i <= loop_num; i++){
                println "loop_num ---> ${i}"
            }
        }
    }
    
}

```

이번에는 외부 주입 숫자로 반복문을 돌려봅시다 좀 익숙한 예약어가 나오는데 우리는 위에서 변수는 전부 def 로 선언을 했지만 이 def 는 아시다 싶히 변수가 들어가는 값에 따라서 데이터 타입이 자동으로 할당됩니다 즉 우리는 이때 int 가 아닌 def loo_num 을 써도 문제가 없는것이죠 하지만 저는 이 loop_num 은 명시적으로 데이터를 할당하기 위한 조치라고 생각해서 int 타입으로 선언을 했습니다 그에 따른 이유는 뒤에 toInteger 가 붙었기 떄문에 이때는 명시적으로 변수를 선언을 해야 겠다는 판단이 들어서 입니다 
물론 def 를 써도 문제가 있는것은 아닙니다 개인의 선택일뿐인거죠 

## 반복문 3

```

stage('loop in String Array'){
    steps {
        script {
            
                def str_array = ["str1" , "str2" , "str3"];    
                for(def temp : str_array){
                    println "${temp}"
                }
        }
        
    }
}

```

이렇게 foreach 와 배열을 선언하는거 까지 우리는 Groovy 를 잠깐 두고 이제 사용할 Groovy 는 그때마다 설명을 하는것으로 진행을 하겠습니다 
보시면 아시겠지만 사실 Groovy 언어는 java 에서 부터 출발한 언어이기 때문에 생각보다 java 랑 유사도가 매우 높습니다 그래서 java 와 호환도도 높고 
java 에서는 못쓰는 동적 할당 (물론 이것은 jdk 11 var 가 출현한 뒤로 java 에서도 가능함) 도 가능했었습니다 Groovy 가 개발된 2003 년에 여전히 java 는 동적 할당이 안되였지만 Groovy 가 출현한 당시에는 java 문법하고 비슷한데 동적이 할당이 된다는 큰 차이점이 있게 됩니다 