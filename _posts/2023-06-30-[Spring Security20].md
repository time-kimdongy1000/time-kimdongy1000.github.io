---

title: Spring Secuirty 20 OAuth2
author: kimdongy1000
date: 2023-06-30 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

우리는 지난시간에 시큐리티의 기본에 대해서 알아보았다 물론 전부다 알아본것은 아니지만 기본적인 로그인 구현 및 방식부터 해서 직접 어플리케이션을 만들어보고
로그인도 해보았다 오늘부터 하게될 이 OAuth2 라는것은 우리들이 생각보다 익숙할것이다 

요즘 웹사이트 로그인할때 보면 보통 그 사이트에서 제공하는 로그인 form 방식이 있는 반면에 구글 로그인 카카오로그인 네이버 로그인 이렇게 대표적인 로그인 

버튼들이 있고 그것들을 통해서 우리는 사이트 가입없이 간편하게 로그인을 할 수 있는데 그러한 기술이 OAuth2 이다 우리는 OAuth2 에 대해서 알아볼려고 한다 

그리고 제일 마지막에서는 구글 로그인 , 네이버 로그인 , 카카오그인을 개발하는 것으로 OAuth2 에 대한 내용을 마무리 하도록 하겠습니다

OAuth2 는 우리가 앞으로 배울 OAuth2 - Login , OAuth2 - Client , OAuth2 - ResourceServer , OAuth2 - Authorization - Sever 
이렇게 큰 기술이 4개나 존재하는데 그중에서 우리는 OAuth2 - Login , OAuth2 - Client 에 대해서만 알아볼 예정이고 

OAuth2 - ResourceServer , OAuth2 - Authorization - Sever 는 좀더 공부가 필요한 부분이다 이에 대해서는 아마 올해 안에는 기술을 안할꺼 같습니다


## 개발환경 

JDK 11 이상 
Spring boot 2.7.1 
keycloak
Window11

## keycloak
일명 인가서버라고 하는 소프트웨어를 설치할것입니다 이를 통해서 우리는 OAuth2 가 무엇인지에 대해서 천천히 공부를 해볼것입니다 인가서버는 우리가 앞에서 시큐리티를 공부할때는 로그인을 구성하고 이 사람의 권한이 무엇이고 이런것들을 전부 우리가 구현을 해주어야 했지만 앞으로 OAuth2 기술을 쓰게 되면 우리는 로그인이 된 이후만 생각하면 되고 로그인 절차 그리고 User 같은 것은 모두 인가서버에서 해결을 해줄것입니다 

1. 소프트웨어 다운로드 주소
<https://www.keycloak.org/archive/downloads-21.0.0.html> 저는 원래 19.1 버전으 사용했는데 이전버전이 다 없어진 관계로 21버전을 설치하겠습니다

원래는 최신버전을 쓸려고 했는데 계속 설치를 해서 안되서 찾아보니 현재 20버전 이상에서는 window 에서 첫번째 실행은 되지만 두번째 실행부터는 안된다는 오류가 발견되었습니다 
무엇인가 특별한 조치가 없는 관계로 원래쓰던 19.1 버전 링크를 남기겠습니다 

<https://github.com/keycloak/keycloak/releases/download/19.0.1/keycloak-19.0.1.zip>

여기서 알집을 받아준뒤 알집을 풀어줍니다 저는 C 에 풀었습니다 (바탕화면에도 상관 없습니다)

2. KeyCloak 실행 
cmd 터미널 열어서 C:\keycloak-19.0.1\keycloak-19.0.1\bin 여기 까지 들어온뒤 

```
.\kc.bat start-dev

```

이렇게 하면 개발모드로 실행이되고 처음 실행하면 버전에 대해서 초기 설정을 진행합니다 (20 버전 이상부터는 이 초기설정에서 계속 오류 발생 구관이 명관...)

3. 접속 
주소가  http://0.0.0.0:8080 나오는데 이는 IPV6 주소이고 이는 http://localhost:8080 으로 접속을 하시면됩니다 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/608aae47-d8d7-4242-8e77-3cd92f1575d5)

그러면 이런 초기 화면이 나오는데 제일 왼쪽에 Administration Console 에 초기 아이디 비밀번호를 입력하겠습니다 

아무것이나 해도 되지만 저는 admin/1234567890 으로 진행하겠습니다 

4. Administration Console 접속 

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f201b70a-fc78-4b76-9dfb-b9f492b1a6e9)

여기에 아까 만들었던 접속계정정보를 입력하겠습니다 

5. Realm 만들기 

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7d76d964-4383-49bb-aecd-6ab722a5857a)

이곳을 클릭하면 파란색 버튼으로 Realm(왕국) 만들 수 있습니다 저희만의 왕국을 만들어보겠습니다 

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/d1c79a26-a7c8-4365-bfaa-11dedc800ffa)

왕국의 이름을 짓겠습니다 저는 Srping-Oauth2-Authorizaion-Project 로 짓겠습니다 

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7473701a-5c51-4eff-b2d1-e418d946f74b)

우리만의 왕국이 만들어지게 됩니다 

6. Client 만들기 

![7](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/6b0f1c37-23ef-4451-b587-81b31075958f)

좌측메뉴에서 Client 선택후 나오는 화면에서 파란색 버튼 Create client 클릭 다음에 나오는 화면에서 이름만 필수값임으로 Spring-Oauth2-Authorizaion-client 게 넣어주고 다음클릭

![8](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/a77769a7-bdfa-416a-8309-f4044a4524e5)

그리고 다음에 나오는 설정에서 이렇게 설정을 하고 넘어갑니다 (뒤에 따로 설정할 수 있습니다)


![9](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/5aee067e-2e1d-4439-a246-6db776467318)

그리고 다음에 나오는 화면에서는 다른건 손대는거 없이 Valid redirect URIs 만 http://localhost:8081/login/oauth2/code/keycloak 설정을 하겠습니다 
이유는 천천히 설명을 할것입니다 

7. User 생성
이 User 는 이제 우리가 앞으로 사용하게될 간편사용자 아이디입니다 

![10](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/5099572c-726b-4455-a9d9-82741bc39479)

왼쪽메뉴에서 user 누르고 create user 클릭한뒤 본인이 원하는대로 설정을 하시면됩니다 그리고 save 누르면 유저가 생성이 됩니다 


## spring 연동 

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
	<artifactId>SpringBoot_web_Security</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>SpringBoot_web_Security</name>
	<description>Project_Amadeus</description>
	<properties>
		<java.version>11</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/org.springframework.security/spring-security-oauth2-client -->
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-oauth2-client</artifactId>
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

기존 maven 에 spring-security-oauth2-client 부분만 추가하겠습니다 

## demoController 만들기 

```

package com.cybb.main.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {
    
    @GetMapping("")
    public String demoController(){
        
        return "demo";
    }
}


```

간단한 demoController 만들어주고 

## application.properties 설정 

```

server.port=8081

spring.security.oauth2.client.registration.keycloak.clientId=Spring-Oauth2-Authorizaion-client
spring.security.oauth2.client.registration.keycloak.clientSecret=NIe2qftuPcclGWFiBFicEWoK5SfYs7ql
spring.security.oauth2.client.registration.keycloak.redirectUri=http://localhost:8081/login/oauth2/code/keycloak
spring.security.oauth2.client.registration.keycloak.scope=email,profile
spring.security.oauth2.client.registration.keycloak.clientName=Spring-Oauth2-Authorizaion-client
spring.security.oauth2.client.registration.keycloak.authorizationGrantType=authorization_code
spring.security.oauth2.client.registration.keycloak.clientAuthenticationMethod=client_secret_post

spring.security.oauth2.client.provider.keycloak.issuerUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project
spring.security.oauth2.client.provider.keycloak.authorizationUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth
spring.security.oauth2.client.provider.keycloak.tokenUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token
spring.security.oauth2.client.provider.keycloak.userInfoUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo
spring.security.oauth2.client.provider.keycloak.jwkSetUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/certs
spring.security.oauth2.client.provider.keycloak.userNameAttribute=preferred_username





```

설정을 해주시면됩니다 이때 중요한것은 clientSecret 인데 

## ClientSecret 확인

![11](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/46f4d4a9-e098-4aa1-884f-ac319cfa8924)

웹에서 이 부분 Client secret 클릭해서 확인후 붙여넣으시면됩니다 나머지는 저랑 

## Client ID 확인
현재 spring.security.oauth2.client.registration.keycloak.clientId , spring.security.oauth2.client.registration.keycloak.clientName 에 들어가는 값으로

![12](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/3f9290dd-3834-404f-b95b-3d5f91ffe297)

여기에 ClientId 있으니 이 부분을 넣어주시면됩니다 그리고 하단 provider 부분은 아마 저랑 realms 를 똑같이 했다면 위랑 같이 가면되는데 그게 아니라면

## provier 값 확인

![13](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/e6812e07-4aec-4cd2-929d-dc21c5d27735)

에서 하단 EndPoint 
OpenID Endpoint Configuration 클릭 그럼 key - value object 가 존재하는데 하나씩 key 를 매핑시켜드리겠습니다 

issuerUri -> issuer
authorizationUri -> authorization_endpoint
tokenUri -> token_endpoint
userInfoUri -> userinfo_endpoint
jwkSetUri -> jwks_uri

이렇게 매핑을 시켜주시면됩니다 단 이것은 저랑 realms 달리했을 경우입니다 같이했으면 저랑 같은거 쓰셔도 됩니다 그럼 준비는 끝 기동하겠습니다 

## localhost:8081/demo 접속

![14](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/2833cb56-515b-42c6-8184-da6fa5717684)

우리가 아는 그 시큐리티 기본 로그인창이 나온다고 생각을 했겠지만 현실은 keyClock 에서 제공하는 로그인창이 뜬것이다 여기에 우리는 아까 넣었던 user 정보를 넣겠습니다 
만약 비밀번호를 설정안해주셨다면

![16](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f0c389dc-0b3d-47c8-8ec0-5e72e17e843c)
이쪽으로 오셔서 다시 설정을 해주시면됩니다 (ResetPassword) 아니면 파란색 버튼으로 비밀번호 만들 수 있는 버튼이 하나 생성이 됩니다 


![15](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/b823b53b-a1f7-4f6f-844f-157df575cf03)

이렇게 넘어오면 성공입니다 

우리는 그러면 개념은 다음시간부터 다루고 처음으로 인가서버와 , spring 애플리케이션을 연동을 해보았습니다 앞에서 우리는 인가서버와 그에 해당하는 리소스 서버를 오롯이 
우리가 구현을 했다면 이 OAuth2-login , client 방식은 이미 구현이 되어 있는 인가서버를 통해서 로그인 후 권한을 획득 한후 권한에 알맞는 페이지로 인도하게 됩니다 

다음시간부터는 이 Oauth2 의 개념부터 차근차근 해보도록 하겠습니다 











