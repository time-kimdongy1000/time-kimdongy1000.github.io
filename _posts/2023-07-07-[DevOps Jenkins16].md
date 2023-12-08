---

title: DevOps Jenkins 실전배포 환경 만들기 2
author: kimdongy1000
date: 2023-07-07 14:00
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

지난시간에 VM 한대를 더 열어서 서버설치 후 기동까지 하는 모습을 보였다 이번시간에는 간단한 어플리케이션을 만든뒤 이를 수동으로 jar 로 만들어서 tomcat 에 넣어준뒤 배포를 해보자 



개발환경 

jdk11 

spring boot 2.7.1

packaging war



```

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.1</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.demo</groupId>
	<artifactId>SpringBoot-Web-Jenkins-Project</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>SpringBoot-Web-Jenkins-Project</name>
	<description>Project_Amadeus</description>
	<packaging>war</packaging>
	<properties>
		<java.version>11</java.version>

	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
			<scope>provided</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>




```
maven 은 이렇게 되고 외부에서 톰캣을 제공하기에 

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>

```

이렇게 둘것이다 

## SpringBootServletInitializer

```

package com.cybb.main;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.servlet.support.SpringBootServletInitializer;

@SpringBootApplication
public class SpringBootWebJenkinsProjectApplication extends SpringBootServletInitializer {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebJenkinsProjectApplication.class, args);
	}

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
		return builder.sources(SpringBootWebJenkinsProjectApplication.class);
	}
}


```

스프링 boot 는 기본적으로 jar 로도 실행을 할 수 있지만 war 로 실행해서 외부 톰캣에 배포하기 위해선 SpringBootServletInitializer 을 상속을 받아야 합니다 
그리고 이처럼 자신의 클래스를 소스로 사용해라는 뜻입니다 

## 간단한 demo 화면 구축

```

package com.cybb.main.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("demo")
public class DemoController {
    
    @GetMapping("")
    public String demoController(){
        
        return "demoController";
    }
}


```
배포를 한뒤에 ip:8080/demo 를 호출하면 demoController 끝 


간단한 프로젝트 작성후 이를 war 로 만들겠습니다 
![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/bda19267-db8e-4097-b07f-2b4b34bee399) 여기에서 package 를 클릭 인텔리j 기준입니다 

그럼 war 가 만들어지는 위치는 /target 에 만들어지게 됩니다 

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/99aad310-efed-4fee-a368-a17fa8e1c70b)

이렇게 war 가 만들어지고 이를 우리가 앞에서 설치한 webapp 에 위치시켜줍니다 

/home/was1/was/webapps 위치에 넣어주시고 

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/dd39e4d2-40dd-4f0e-95c7-5ad1d67f1100)

이 위치가 될것이다 

## server.xml 

vi /home/was1/was/conf/server.xml

```

<Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

    <Context docBase="SpringBoot-Web-Jenkins-Project-0.0.1-SNAPSHOT" path="/" reloadable="false"/>

</Host>




```

하단에 위와 같이 Context 를 적어주면된다 이때 나랑 다른 프로젝트 명이라면 자신의 프로젝트 명으로 만들어주면된다 

## 기동 

기동은 위에서 보았듯이 bin/startup.sh 을 실행해주면 된다 

## demoController 요청

![7](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/3e924d5f-8b24-4bb7-9e14-d337fc553c17)


이렇게 ip:8080/demo 로 했을때 이렇게 나오면 배포된 파일을 올바르게 본것이다 

이렇게 우리는 수동으로 tomcat 에 배포하는 것을 공부했다 다음시간 부터는 이제 이들을 자동으로 배포 할 수 있게 파이프라인 스크립트로 작업을 해볼 예정입니다 