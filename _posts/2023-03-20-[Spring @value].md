---
title: Spring Ioc @Value
author: kimdongy1000
date: 2023-03-20 14:20
categories: [Back-end, Spring - Core]
tags: [ "@value" ]
math: true
mermaid: true
---

## @Value 
스프링 프레임워크에서 제공하는 어노테이션 중 하나로, 주로 프로퍼티 값을 주입하기 위해 사용됩니다. @Value 어노테이션을 사용하면 XML이나 JavaConfig와 같은 외부 설정 파일에서 값을 읽어와 빈에 주입할 수 있습니다. 

우리는 앞에서 이미 @Value 를 한번 본적이 있습니다 기억을 더듬어보면 다중DB 연결할때 사용했습니다 

## 다중DB 연결 일부분
```

@Value("${oracle.datasource.url}")
private String url;

@Value("${oracle.datasource.username}")
private String username;

```
다중 DB 연결할때 우리는 이렇게 `${}` 형식을 주고 개발을 외부에서 갓을 주입을 했습니다 이때 이 안에 들어가는 값들은 주입해줄 값의 key 값입니다 즉 (oracle.datasource.url) key 값 그러면 외부값은 이 key 값을 바탕으로 값을 찾아오게 됩니다 

## application.properties 
```
oracle.datasource.url=jdbc:oracle:thin:@192.168.46.129:1521:ORCL
oracle.datasource.username=kimdongy1000

```
그럼 그 외부값의 대표적인게 바로 application.rpoperties 입니다 key - value 형태로 되어 있으며 spring 이 기동될때 이 key 값을 찾아서 @Value 안에 주입을 해주게 됩니다 



## 외부파일의 분리 
조금더 규모있는 프로젝트로 가면 각 conifg 마다 파일 명을 분리를 해놓게 됩니다 지금은 application.properties 에 모든 정보를 다 집어 넣었다면 이번에는 다음과 같은 방식으로 한번 변경을 해보겠습니다 

파일 위치는 이렇게 될 예정이다 

-project 
	-src
		-main
			-java
				-com
					-cybb
						-main
							-config
								-MysqlConfig.java
								-OracleConfig.java
							-repository
								-MysqlRepository.java
								-OracleRepository.java
						resource
							-static
								-mapper 
									-multiple
										-mysql
											-mysql.xml
										-oralce								
											-oracle.xml
						-application.properties
						-mysql_db.properties
						-oracle_db.properties

지금 우리는 application.properties 파일에서 각 db 정보를 2개의 properties 로 옮겨서 진행을 할것이다 지금 당장 properties 만 분리를 하게 된다면 이런에러를 보게 될것이다 

```

Could not resolve placeholder 'oracle.datasource.url' in value "${oracle.datasource.url}

```
지금 상태에서 oracle.datasource.url 이라는 key 값을 찾지 못해서 값을 주입할 수 없다면서 에러가 발생한다 즉 기본적으로 @Value 가 값을 읽어오는 위치는 아무런 설정을 하지 않게 되면 applicaiton.properties 이다 그럼 저기 두개파일을 읽게 할려면 다음과 같은 설정을 해주어야 한다 

## @PropertySource
```
@Configuration
@PropertySource("mysql_db.properties")
public class MysqlConfig {


}

@Configuration
@PropertySource("oracle_db.properties")
public class OracleConfig {


}

```
@PropertySource 어떤 특정한 파일을 classpath 에 로드하여 프로퍼티 소스로 사용하게끔 하는 애노테이션중 하나입니다 즉 기본적인 properties 읽어오는 위치는 application.properties 인데 이를 사용하게 되면 보다 구체적인 위치를 명시해서 불러올 수 있습니다 그리고 @PropertySource 는 기본위치는 클래스 path 이지만 실제로는 파일path 위치에도 쓸 수 있습니다 

예를 들어서 DB 같은 정보를 가리고 싶다 특정정보는 가리고 싶을때에는 이런식으로 사용하고 지금보이는 프로젝트 상에서는 지울 수 있습니다 

그럼 어떻게 되느냐 

-project 
	-src
		-main
			-java
				-com
					-cybb
						-main
							-config
								-MysqlConfig.java
								-OracleConfig.java
							-repository
								-MysqlRepository.java
								-OracleRepository.java
						resource
							-static
								-mapper 
									-multiple
										-mysql
											-mysql.xml
										-oralce								
											-oracle.xml
						-application.properties

디렉토리 상에서는 기존의 -mysql_db.properties , -oracle_db.properties 없어지게 되고 이를 jar 로 말아서 서버에 배포된 상태라면 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/37539c75-7490-4cd0-a3f7-faa27838495b)

이런 모양으로 배포 되었다고 하자 

-home
	-kimdongy1000
		-springProject
			-SpringBoot_web_Security-0.0.1-SNAPSHOT.jar
			-config
				-mysql_db.properties
				-oracle_db.properties

이런모양이다 즉 기존의 properties 가 이제 완전히 바깥으로 나간 모습을 볼 수 있다 이때도 spring 의 @PropertySource 파일 시스템상에 있는 파일또한 읽어서 주입을 할 수 있습니다 

## File 시스템에서 읽어오기 
```

@Configuration
@PropertySource("file:/home/kimdongy1000/springProject/config/mysql_db.properties")
public class MysqlConfig {


}

@Configuration
@PropertySource("file:/home/kimdongy1000/springProject/config/oracle_db.properties")
public class OracleConfig {


}

```

이렇게 file 이라는 prefix 를 주게 되면 이제 spring 은 이 두개의 파일을 현재 배포된 파일 시스템 안에서 찾게 됩니다 그래서 완전히 숨기고 싶은 외부주입 같은 것들은 이렇게 해서 
스프링 시스템 안으로 불러오게 할 수 있습니다