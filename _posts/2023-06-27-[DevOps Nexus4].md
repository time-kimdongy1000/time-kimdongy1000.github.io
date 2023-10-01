---

title: DevOps Nexus 4 나만의 저장소에 나만의 라이브러리 올리기 
author: kimdongy1000
date: 2023-06-27 16:30
categories: [DevOps, Nexus]
tags: [ Nexus ]
math: true
mermaid: true

---

지난시간에 우리는 나만의 저장소를 만들었고 나만의 그룹에 maven - central 를 같이 묶어서 그룹으로 만드는거 까지 진행했습니다 오늘은 

나만의 저장소에 라이브러리에 올리는 작업을 진행하겠습니다 그 전에 Maven 에 대해서 알아보도록 하겠습니다

## maven 의 생명주기

1. validate: 프로젝트 구조와 파일 구조가 올바른지 확인합니다. 이는 빌드하기 전에 프로젝트의 구성을 검증하는 단계입니다.

2. compile: 프로젝트의 소스 코드를 컴파일하여 바이트 코드로 변환합니다.

3. test: JUnit과 같은 테스트 프레임워크를 사용하여 프로젝트의 테스트를 실행합니다.

4. package: 컴파일된 코드와 필요한 리소스를 묶어서 JAR, WAR, 또는 EAR와 같은 패키지로 만듭니다.

5. verify: 패키지가 올바른지 확인합니다. 예를 들어, 패키지에 손상된 파일이나 중복된 파일이 있는지 확인합니다.

6. install: 패키지를 로컬 리포지토리에 설치합니다. 다른 프로젝트에서 이 패키지를 사용할 수 있습니다.

7. deploy: 패키지를 원격 리포지토리에 배포합니다. 이 단계는 일반적으로 라이브 서버에 코드를 배포할 때 사용됩니다.

maven 을 실행할때 이와 같은 생명주기로 움직이며 모든 생명주기를 실행하기 전에는 clean 을 사용하게 되는데 clean 은 앞의 결과를 지우는 역활을 하게 됩니다 

그리고 상위 생명주기를 실행하면 하위 모든 생명주기가 실행이 됩니다 

maven 명령어로 clean install 을 실행하게 되면 clean -> validate -> compile -> test -> package -> verify -> install 이렇게 하단에 있는 모든 실행과정을 거치게 됩니다 

그리고 제일 많이 사용하는 것은 보통 clean , package , install , deploy 를 가장 많이 사용하게 됩니다 이 명령어들은 보통 해당 코드를 라이브러리로 패키징 하게 됩니다


## settings.xml 수정 

```

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">

<localRepository>C:\Users\kimdo\.m2\repository</localRepository>

<profiles>
        <profile>
                <repositories>
                        <repository>
                                <id>nexus</id>
                                <url>http://192.168.46.131:8081/repository/maven-central/</url>
                                <releases>
                                        <enabled>true</enabled>
                                </releases>
                                <snapshots>
                                        <enabled>true</enabled>
                                </snapshots>
                        </repository>
                        
                        <repository>
                                <id>nexus2</id>
                                <url>http://192.168.46.131:8081/repository/time-maven-hosted3A/</url>
                                <releases>
                                        <enabled>true</enabled>
                                </releases>
                                <snapshots>
                                        <enabled>true</enabled>
                                </snapshots>
                        </repository>
                </repositories>

                <pluginRepositories>
                        <pluginRepository>
                                <id>nexus</id>
                                <url>http://192.168.46.131:8081/repository/maven-central/</url>
                                <releases>
                                        <enabled>true</enabled>
                                </releases>
                                <snapshots>
                                        <enabled>true</enabled>
                                </snapshots>
                        </pluginRepository>
                </pluginRepositories>

        </profile>
</profiles>


<mirrors>
        
        <mirror>
                <!--This sends everything else to /public -->
                <id>nexus2</id>
                <url>http://192.168.46.131:8081/repository/time-maven-hosted3A/</url>
                <mirrorOf>*</mirrorOf>
        </mirror>
        
        <mirror>
                <!--This sends everything else to /public -->
                <id>nexus</id>
                <url>http://192.168.46.131:8081/repository/maven-central/</url>
                <mirrorOf>central</mirrorOf>
        </mirror>
        
</mirrors>
    





<servers>
    
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>######</password>
    </server>
    
      <server>
      <id>nexus2</id>
      <username>admin</username>
      <password>######</password>
    </server>

</servers>






</settings>




```

우리는 이렇게 여러개의 repository 를 mirror 태그로 여러개 구분할 수 있습니다 그리고 전체적으로 settings.xml 을 수정했습니다 


## 간단한 라이브러리 파일 만들기 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/95328926-c827-4cd0-b26a-7feee748e5ec) 

우리는 간단한 maven 프로젝트로 간단한 라이브러리 파일을 만들어보겠습니다 결론적으로 이 라이브러리 nexus 파일로 배포를 하고 다른 프로젝트에서 이 라이브러리르 쓰는것을 보여주겠습니다

```

package com.time.timecommon.model;

public class User {
    
    private String username;
    
    private String password;
    
    private boolean isAuthenticated;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public boolean isAuthenticated() {
        return isAuthenticated;
    }

    public void setAuthenticated(boolean authenticated) {
        isAuthenticated = authenticated;
    }
}


```

패키지 안에 간단한 모델을 만들어보겠습니다 

## pom.xml 수정 

```

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.time</groupId>
  <artifactId>timecommon2</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <name>timecommon2</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <java.version>11</java.version>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>
  
  <distributionManagement>
    <repository>
      <id>nexus2</id>
      <name>Releases</name>
      <url>http://192.168.46.131:8081/repository/time-maven-hosted3A/</url>
    </repository>
  </distributionManagement>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>



```

pom.xml 에 distributionManagement 태그가 추가 되었습니다 이때 id 와 url 을 입력하게 되면 이는 앞에서 입력한 settings.xml 

```

<mirror>
        <!--This sends everything else to /public -->
        <id>nexus</id>
        <url>http://192.168.46.131:8081/repository/maven-central/</url>
        <mirrorOf>central</mirrorOf>
</mirror>

<server>
      <id>nexus2</id>
      <username>admin</username>
      <password>########</password>
    </server>

```

이 서버 비밀번호를 인식해서 nexus 에 자동으로 배포를 하게 됩니다 

## deploy 생명주기 실행 

그럼 이 라이브러리를 nexus 에 올리도록 하겠습니다 생명주기 deploy 를 사용하면되는데 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/566f8653-c1f1-49aa-ae85-4f39562c240b)

이렇게 옆에 화면을 열고 deploy 를 더블클릭하면 생명주기가 실행이 되고 하단에 현재 상태가 나오게 되고 BUILD SUCCESS 가 뜨면 완료가 된것입니다 

그러면 

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/af024e24-08fb-4651-b89c-f0e4dd3bd43a)

이렇게 나의 개인 리포지토리에 내용이 올라가게 됩니다 이렇게 우리는 우리가 만든 라이브러리를 올리는데 까지 성공했고 이제 서로 다른 프로젝트에 이 라이브러를 다운로드 
받아서 쓰는 모습을 보여주겠습니다

## 사용방법

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/82dffdc3-117b-4745-87a0-c58d9f5d8867) 

모든 라이브러리는 이렇게 우측에 사용하는 방법이 나오게 됩니다 이때는 maven 뿐만 아니라 gradle 등 여러가지 버전으로 사용할 수 있습니다
저는 메이븐으로 사용하겠습니다


```

package com.example.demo;

import org.springframework.boot.SpringApplication;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import com.time.timecommon.model.User;

@SpringBootApplication
public class SpringWebResourceServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringWebResourceServerApplication.class, args);
		
		User user = new User();
		user.setUsername("time");
		user.setPassword("1234567890");
		user.setAuthenticated(true);
		
		System.out.println(user.toString());
			
	}

}


```

결과 

```
...

User [username=time, password=1234567890, isAuthenticated=true]

...

```

완전히 다른 프로젝트에서 우리가 받은 maven 을 사용할 수 있습니다 우리는 이렇게 메이븐 프로젝트를 생성하고 이 내용을 nexus 에 배포를 해서 진행을 했고 
이를 완전히 다른 프로젝트에서 다운로드 받아서 개발을 진행했습니다 




