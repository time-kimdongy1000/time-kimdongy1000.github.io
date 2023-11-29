---
title: Spring Secuirty 31 Resource Server
author: kimdongy1000
date: 2023-07-09 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

우리는 앞에서 OAuth2Login 과 OIDC 에 대해서 공부를 해보았다 이 부분은 주로 클라이언트와 관련된 내용이었다 인증 / 인가와 관련된 내용을 뒤로 하고 오늘 부터는 
OAuth2 Resource Server 에 대해서 공부를 진행을 할 것이다 

앞으로 KeyClock 를 ResourceServer 에 어떻게 사용하는지에 대해서 알아보자 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f40ab778-2504-48d6-a84a-decdceeb4453)

앞으로 이런 모양이 될것이다 

1) 클라이언트는 리소스 서버에 접근해서 제한된 자원을 얻을려고 한다 

2) 이떄 리소스 서버는 클라이언트가 access_token 을 들고오는지 검사한다 

3) 이때 access_token 이 없으면 이를 돌려서 access_token 을 발급받게끔 유도한다 (401 에러)

4) 클라이언트가 access_token 을 가져와서 제한된 자원을 얻을려고 한다면 클라이언트는 이 access_token 을 자체 분석할 수 있게 인가서버에 공개키를 요청 또는 토큰을 직접 보낸다 

5) 공개키를 발급받으면 리소스 서버에서 자체 검증후 토큰이 유효하면 해당 자원을 access 할 수 있게 빗장을 풀어주고 
   만약 자체 검증 토큰이 아니면 이를 인가서버로 전송한뒤 인가서버가 해당 토큰의 유효성을 검증해서 리소스 서버로 전달을 해준다 

6) 그렇지 않으면 마찬가지로 반려시키고 403 에러를 보여준다 

이게 우리가 앞으로 길게 배울 리소스 서버의 특징이다 그럼 리소스 서버를 어떻게 연동을 하는지에 대해서 알아보자 

마찬가지로 바로 소스를 살펴보자 


개발스펙은 

jdk 11 
spring boot 2.7.1 
keyclock 19.0.1


## maven 설정
```
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
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
	</dependency>
</dependencies>
```

maven 에서 알 수 있다 싶히 spring-boot-starter-oauth2-resource-server 의존성을 주입받게 되면 이제 이 프로젝트는 리소스 서버에 관련한 설정을 자동으로 진행을 하게 된다 

## application.properties 

```
server.port=8082

spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project
spring.security.oauth2.resourceserver.jwt.jwkSetUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/certs

```

서버 포트는 이렇게 변경을 하고 나머지는 이렇게 적어준다 이에 대한 설명은 다음시간에 하도록 하자

## demoController
```
@RestController
public class DemoController {

    @GetMapping("")
    public String demoController(){return "demonController";}
}
```

인가를 받은 클라이언트는 여기에 접근해서 demoController 을 얻을것이다 

그럼 기동을 하자 그럼 우리 앞에서 생각을 해보자 Oauth2 연동을 하게 되면 접근시 Oauth2 로그인 화면이 기본적으로 보여지는것을 알 수 있을것이다 하지만 
지금 이 상태에서는 웹에 접근해서 핸들러를 접근을 해도 401 에러만 발생하게 된다 이게 리소스 서버의 특징이다 

앞에서 Oauth2 로그인 설정에서는 인가되지 않은 사용자는 기본적으로 인증 및 인가를 할 수 있게 로그인 창을 기본적으로 제공을 하였지만 
리소스 서버는 그렇지 않다 이미 이 핸들러에 접근할때는 인증 및 인가가 되어 있다고 생각을 하고 진행을 하게 된다 그래서 우리는 access_token 을 리소스 서버에 던져서 
demoController 에 접근을 해보자 이때 클라이언트는 post-man 을 사용할것이다 

## 임시코드 발급

```

http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak

```
postman 에 이 주소를 입력하게 되면 자동으로 파라미터를 세팅하게 되는데 이 주소는 우리가 앞에서 OAuth2AuthorizationRequestRedirectFilter 에 의해서 만들어지는것을 우리는 보았습니다 자세한 내용은 <https://time-kimdongy1000.github.io/posts/Spring-Security24/> 참조해주시면됩니다 

위의 주소로 post - man 을 실행을 하게 되면 

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/55000b52-d8fc-4759-910e-9f00a3026c8c)

이렇게 하단의 response 에 이렇게 나오게 된다 잘 보면 `<title>Sign in to Srping-Oauth2-Authorizaion-Project</title>` 를 볼 수 있는데 이 주소를 
크롬창에 입력을 해서 엔터를 하게 되면 

우리가 잘 보던 로그인 화면이 나오게 된다 여기서 로그인을 해보자 그러면 조금 로딩이 되다가 404 에러가 되는데 에러가 되는것은 당연히 redirect_uri 가 없는이유때문이고 

```
http://localhost:8081/login/oauth2/code/keycloak?state=KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4%3D&session_state=63555d9a-373f-4cc6-b437-7b9e9ab3a835&code=a563135e-299d-468e-8cfa-04924311afd5.63555d9a-373f-4cc6-b437-7b9e9ab3a835.4f8d60ae-7370-4bef-a356-95620f098f06

```

주소 스펙을 잘 살펴보면 code= ~~ 보일것이다 이게 임시코드이다 이를 복사해서 다음 access_token 발급으로 가보자

## access_token 

```

http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token

```

access_token 은 post 로 요청을 주어야 한다 그래서 파라미터는 이와 같은데 

grant_type - authorization_code
client_id - Spring-Oauth2-Authorizaion-client
client_client_secret - NIe2qftuPcclGWFiBFicEWoK5SfYs7ql(본인 keyclocak 의 코드를 입력해주세요)
redirect_uri - http://localhost:8081/login/oauth2/code/keycloak
scope - email%20profile
state - KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4%3D
`code - a563135e-299d-468e-8cfa-04924311afd5.63555d9a-373f-4cc6-b437-7b9e9ab3a835.4f8d60ae-7370-4bef-a356-95620f098f06`

이렇게 입력을 해주세요 

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/62dee1ef-1cc8-437b-a26e-5620981eedd5)

그럼 이렇게 아래에 access_token 과 resource 토큰이 발급되는것을 확인할 수 있습니다 코드의 유효시간은 짧기 때문에 되도록 빨리 입력을 해주셔야 합니다 

## access_token 으로 resource 서버 접근

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/8c00376f-ae11-4c10-b8cb-55e31f1cc663)

이렇게 핸들러 적고 post - man 에 Authorization 토큰으로 Bearer Token 으로 아까 받은 access_token 을 입력하고 send 하면 이렇게 
demoController 이 나오게 됩니다 


그럼 우리는 다음시간부터 어떻게 해서 access_token 을 이용해서 리소스 서버에 접근할 수 있는지에 대해서 공부를 계속해보도록 하겠습니다 