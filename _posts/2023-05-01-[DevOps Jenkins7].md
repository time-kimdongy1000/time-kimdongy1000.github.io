---

title: DevOps Jenkins 변수 및 파라미터 설정
author: kimdongy1000
date: 2023-05-02 15:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

지난시간 까지 jenkins 구조에 대해서 확인을 했고 이번시간부터는 본격적으로 jenkins 빌드할때 구체적으로 파라미터는 어떻게 활용이 되는지 확인을 하겠습니다 그리고 저의 최종목표는 
파이프라인으로 구축해서 완전자동화 프로세서를 만드는게 목표입니다 또 천천히 나가도록 하겠습니다 

## 글을 쓰기 앞서서 

지금으부터 제가 앞으로 쭈욱 완전 자동화배포에 관련한 jenkins 글을 작성할건데 이는 제가 실무에서 경험한거 , 스스로 공부하면서 찾은거 그리도 앞으로 사용하고 싶은 기술을 총 집합해서 
글을 쓸 예정입니다 그러다 보니 jenkins 의 정석보다는 엇나가는 글 그리고 저의 생각이 많이 비춰질 수 있습니다 다만 그런 내용은 최대한 jenkins 의 지향점에 맞춰서 개발될 예정이고 
저도 정제된 글을 쓸려고 노력할것입니다 

## 스크립트 
우리는 파이프라인으로 구축하기 앞서서 단일 스크립트로 먼저 단일 빌드 환경을 먼저 구성해서 배포하고 서버 재기동까지 하는 방식에 대해서 공부를 해보겠습니다 
단순 스크립트로만 구성하는 것을 단일 파이프라인이라고 하는 사람도 있지만 사실 단일 스크립트로 구성되는것들은 파이프라인하고 거리가 멉니다 난중 뒤에서는 파이프라인을 구축하는 시간도 가질 예정입디만 단순 스크립트로만 구성하는것 하고는 다른 수준입니다 

## 새로운 WebProject 생성 
제가 경험한 바로는 Spring-Boot 의 경량 web - was 로만 구동하는 프로젝트는 경험해보지 않았습니다 작게는 오픈소스인 톰캣과 ngingx 로만 구현되어 있는 프로젝트 
크게는 was 로만 웹로직 (이떄당시 web 을 뭘 썼는지 기억이 안남) 쓰는곳을 경험을 해보았습니다 다만 우리가 오늘 만들 프로젝트는 spring boot 안에 tomcat 이 내장이 되어 있는 프로젝트를 경험할 것입니다 그리고 점점 스케일을 키워서 tomcat 에 배포하고 ngignx 로 기동을 하는거 까지 진행을 할 예정입니다 

그럼 저는 간단한 spring boot 프로젝트를 만들겠습니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7a229d25-7cf8-4be4-bdf3-7e6ef4005067)

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0ea2f040-9d26-4405-a36a-af5f40727824)

이렇게 간단하게 web 만 추가해서 진행하도록 하겠습니다 


```

package com.cybb.main;


import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {
	
	@GetMapping("/test")
	public String test() {
		
		return "테스트 페이지 입니다";
	}

}


```

소스는 간단하게 만들고 그리고 port 변경을 하겠습니다 jenkins 안에서 같이 웹was 를 기동시키기에 jenkins 가 사용하는 port 8080은 사용할 수 없기에 8081 로 하겠습니다 

application.properties

```

server.port=8081


```

이렇게 만들고 이를 git 에 올리도록 하겠습니다 저는 앞에서도 계속 말씀드렸다 싶히 gitlab 활용할 예정입니다 그리고 job 하나를 생성하겠습니다 
제목은 원하는대로 만드셔도 되는데 저는 Hello_Spring_boot_Deploy_Project 이렇게 만들도록 하겠습니다 
그리고 maven Proejct 로 만들어주겠습니다 

그리고 지난시간에 한것처럼 

![Hello_Spring_boot_Deploy_Project-Config-Jenkins-](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/a20cc594-15dc-4db4-a911-33e1eeddb71e)

여기 까지는 이제 충분히 만드실수 있을듯 합니니다 그리고 빌드 버튼을 눌러서 빌드를 진행해보겠습니다 마찬가지로 마지막에 빌드 되었습니다 나오고 success 가 뜨면 성공입니다 
다만 앞의 간단한 maven 프로젝트하고는 다르게 엄청난량의 로그가 쌓이는 것을 볼 수 있을것입니다 

그리고 우리는 어제 본 디렉터리에 workspace 에 가면 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/8c8cc4d9-5858-4dcd-8659-686ba14280b0)

이렇게 우리가 만든 프로젝트가 .jar 로 변경되어 있는것을 볼 수 있습니다 여기 까지 오는게 지난번 포스트에 목적이고 우리는 이 기본 프로젝트에 여러가지 
재미있는 기능들을 활용해보겠습니다 

## 오래된 빌드 삭제

기본적으로 모든 jenkins 빌드는 어제 디렉토리 분석에서 보았듯이 다 빌드 이력을 남기게 됩니다 하지만 그렇게 삭제 없이 모든 빌드건들을 남기기 보다는 오래된 빌드 삭제를 통해서 
주기적으로 삭제가 되게끔 해주는것이 좋습니다 

상단에 클릭해보시면 아시다 싶히 오래된 빌드 삭제 체크하게 되면 
빌드 이력 유지 기간  , 보관할 최대갯수 두가지가 나오는데 어느 하나만 적을 수 있고 둘다 적을 수 있습니다 예를 들어서 빌드 이력 유지기간이 10일로 두고 보관할 최대갯수를 100으로 잡으면 
10일 미만이라도 100을 넘개되는 순간 오래된 빌드건들은 삭제되게 됩니다 이는 업체에서 빌드 히스토리를 어떻게 가져갈 것인가에 따라서 달라지게 됩니다 
우리는 기본적으로 10일 10개 로 두겠습니다 

## 이 빌드는 매개변수가 있습니다 

이 부분이 아마 스크립트 배포관련해서 가장 중요한 부분입니다 프로그래밍으로 치자면 변수를 선언하고 가져다 쓸 수 있는 그런곳입니다 우리는 여기에 일단 어떤 내용을 적을것이냐면 
하단에 Post Steps 에 Execute Shell 에 사용할 변수 2개를 선언할것인데 지금은 크게 의미가 없는 변수를 쓰고 어떻게 불러다 쓰는지만 보겠습니다 

하나는 String Parameter 그리고 다른 하나는 Password Parameter 두개를 선언하겠습니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/44595a6f-6485-4ba4-a35f-01e7a699867e)

그럼 이와 같이 될것인데 
이렇게 되면 변수는 다음과 같이 선언됩니다 

```

String HOST = "192.168.40.132"
String password = "1234567890"

```

이렇게 선언될것입니다 다만 password parameter 는 지금처럼 마스킹 처리 된 채로 보이게 됩니다 
그럼 위에서 선언한 변수를 어떻게 가져다 쓰는가 


제일 하단 Post Step 으로 와서 
Execute shell 로 와서 

```

#!/bin/bash 
	
echo "빌드 되었습니다"

echo "HOST : ${HOST}"
echo "password : ${password}"

```

이렇게 됩니다 자 그럼 이걸 저장하고 다시 빌드하기 클릭하면 우리가 못보던 화면 하나가 더 등장합니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/cc1b9fa8-b056-489a-a8a7-6b0c1dc124f9)

이렇게 말이죠 이뜻은 우리는 build 할때 기본 매개변수를 설정한것입니다 그래서 기본으로 설정한 값들이 채워서 보이게 되고 빌드를 할때 얼마든지 이 부분을 수정해서 
빌드할 수 있습니다 단 host 주소나 password 주소는 수정되서는 안되는 정보이기 때문에 보통 이런곳에 심지는 않습니다 

어찌되었든 지금은 이런 변수들이 어떻게 사용이 돠나 보여드리기 위함입니다 다시 초록색 빌드하기 버튼을 클릭하게되면 


```

[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  9.668 s
[INFO] Finished at: 2023-07-26T14:31:36+09:00
[INFO] ------------------------------------------------------------------------
[WARNING] 
[WARNING] Plugin validation issues were detected in 5 plugin(s)
[WARNING] 
[WARNING]  * org.apache.maven.plugins:maven-resources-plugin:3.2.0
[WARNING]  * org.apache.maven.plugins:maven-jar-plugin:3.2.2
[WARNING]  * org.apache.maven.plugins:maven-install-plugin:2.5.2
[WARNING]  * org.apache.maven.plugins:maven-compiler-plugin:3.10.1
[WARNING]  * org.apache.maven.plugins:maven-surefire-plugin:2.22.2
[WARNING] 
[WARNING] For more or less details, use 'maven.plugin.validation' property with one of the values (case insensitive): [BRIEF, DEFAULT, VERBOSE]
[WARNING] 
Waiting for Jenkins to finish collecting data
[JENKINS] Archiving /var/lib/jenkins/workspace/Hello_Spring_boot_Deploy_Project/pom.xml to com.demo/SpringBoot-Web-Jenkins-Project/0.0.1-SNAPSHOT/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.pom
[JENKINS] Archiving /var/lib/jenkins/workspace/Hello_Spring_boot_Deploy_Project/target/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar to com.demo/SpringBoot-Web-Jenkins-Project/0.0.1-SNAPSHOT/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar
channel stopped
[Hello_Spring_boot_Deploy_Project] $ /bin/bash /tmp/jenkins7822355453958288830.sh
빌드 되었습니다
HOST : 192.168.40.132
password : 1234567890
Finished: SUCCESS


```

그럼 우리는 위에서 설정한 변수를 아래에서 사용한것을 보았습니다. 

## ${WORKSPACE} 

jenkins 에도 예약어 변수가 몇가지 존재하는데 그중 내가 잘 사용하는 시스템 변수는 ${WORKSPACE} 이다 이 변수는 현재 jenkins 가 어디에서 작업을 하고 있는지 변수에 넣게 되는데 다음과 같이 적어보자 

```

#!/bin/bash 

echo "빌드 되었습니다"


echo "HOST : ${HOST}"
echo "password : ${password}"
echo "WORKSPACE : ${WORKSPACE}"

```

이렇게 적고 빌드를 해보면

```

[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  8.544 s
[INFO] Finished at: 2023-07-26T14:36:59+09:00
[INFO] ------------------------------------------------------------------------
[WARNING] 
[WARNING] Plugin validation issues were detected in 5 plugin(s)
[WARNING] 
[WARNING]  * org.apache.maven.plugins:maven-resources-plugin:3.2.0
[WARNING]  * org.apache.maven.plugins:maven-jar-plugin:3.2.2
[WARNING]  * org.apache.maven.plugins:maven-install-plugin:2.5.2
[WARNING]  * org.apache.maven.plugins:maven-compiler-plugin:3.10.1
[WARNING]  * org.apache.maven.plugins:maven-surefire-plugin:2.22.2
[WARNING] 
[WARNING] For more or less details, use 'maven.plugin.validation' property with one of the values (case insensitive): [BRIEF, DEFAULT, VERBOSE]
[WARNING] 
Waiting for Jenkins to finish collecting data
[JENKINS] Archiving /var/lib/jenkins/workspace/Hello_Spring_boot_Deploy_Project/pom.xml to com.demo/SpringBoot-Web-Jenkins-Project/0.0.1-SNAPSHOT/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.pom
[JENKINS] Archiving /var/lib/jenkins/workspace/Hello_Spring_boot_Deploy_Project/target/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar to com.demo/SpringBoot-Web-Jenkins-Project/0.0.1-SNAPSHOT/SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT.jar
[Hello_Spring_boot_Deploy_Project] $ /bin/bash /tmp/jenkins17875531757401777422.sh
channel stopped
빌드 되었습니다
HOST : 192.168.40.132
password : 1234567890
WORKSPACE : /var/lib/jenkins/workspace/Hello_Spring_boot_Deploy_Project
Finished: SUCCESS

```

WORKSPACE : /var/lib/jenkins/workspace/Hello_Spring_boot_Deploy_Project 현재 어디에서 작업을 진행하고 있는지 변수에 저장이 되어 있고 이것을 아주 유용하게 가져다 쓸 수 있다 
이러한 변수 활용만 잘 알아도 우리는 jenkins 의 배포된 내용을 다른 곳으로 바로 던질 수 있는 환경을 만들 수 있다 

이에 대한것은 다음 포스트에서 알아보자 





















