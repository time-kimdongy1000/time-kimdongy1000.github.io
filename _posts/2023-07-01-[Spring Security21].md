---

title: Spring Secuirty 21 OAuth2 기본개념 
author: kimdongy1000
date: 2023-07-01 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

우리는 지난시간에 아무개념도 모른채 KeyClock 를 설치하고 SpringBoot 연동을 해서 로그인하는거 까지 살펴보았다 당분간은 로그인이 되는 과정보다는 먼저 OAuth2 에 대한 개념설명을 오늘 먼저 할려고 합니다 

개념설명은 무엇보다 용어정리가 우선이 되어야 합니다 

## 용어정리 

1. ReSource Owner 
이는 보호된 자원에 대한 접근을 최종적으로 승인하는 주체 (즉 로그인하는 자신을 말합니다) 이때 보호된 자원은 나의 프로필 (이름을 포함한 생년월일 휴대전화 번호 등등)
우리가 KeyClocak 이 제공하는 로그인 Form 을 이용하여 로그인을 진행했을때 ReSource Owner 는 나 (자신) 를(을) 뜻하게 됩니다 

2. ReSource Server 
사용자의 자원이 포함된 서버 KeyClock 같은 경우는 인가서버 (Authorizaion - Server) 과 자원 서버 (ReSource - Server) 이 같이 있는 경우입니다 
인가서버는 로그인을 할떄 사용했고 자원서버는 우리가 로그인을 하고 (실제 사용하지 않았지만 우리가 생성한 User 를 시큐리티가 이용할 수 있음) 특정한 행동을 할때 사용이 됩니다 

3. Authorization Server 
클라이언트가 사용자 계정에 대한 동의 및 접근을 요청할때 사용하는 서버로 클라이언트의 사용자 인증을 승인하거나 거부하는 서버 
(KeyClocak 엔진 자체)

4. Client 
사용자의 자원을 엑세스 할려는 어플리케이션 (Spring boot - demoController)


## 원리 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/93340445-56a2-4ff3-900e-b6a68fefedb4)

대략 큰 틀에서는 이런 형식으로 돌아갈것이다 

이전에 클라이언트에서 로그인 처리를 담당했던 부분을 인가서버로 책임을 전달해서 그 사람에 대한 정보 및 허용된 사람인지 여부는 인가서버에서만 처리 
대신 클라이언트는 인가서버로 부터 받은 정보를 신뢰하며 허가된 사람이면 최초 자원 소유자의 요청에 응하는 이런식입니다 

Oauth2 는 큰틀에서 이렇게 움직입니다 

계속해서 용어정리를 이어나가겠습니다 

## application.properties 

우리는 지난시간에 keyClock 을 연동할때 연동정보를  application.properties  저장을 해두었다 이곳에도 알아야할 정보들이 있으니 차근차근 공부를 해보자

1. clientId  (clientId=Spring-Oauth2-Authorizaion-client)
이는 인가서버에서 클라이언트를 식별할때 사용하는 유일한 Key 값입니다 즉 우리가 만든 DemoController 의 Client Id 및 name 은 Spring-Oauth2-Authorizaion-client 가 되는 것입니다 

2. clientSecret (clientSecret=NIe2qftuPcclGWFiBFicEWoK5SfYs7ql)
이는 clientId 와 마찬가지로 clientId에 대한 비밀Key 입니다 이 Key 를 활용해서 인가서버와 통신할때 ClientId 와 같이 제출함으로서 이 ClientId 의 요청이 인가된 애플리케이션에서 요청이 들어왔는지 확인하게 됩니다 

3. redirectUri (redirectUri=http://localhost:8081/login/oauth2/code/keycloak)
사용자가 인가서버에 성공적으로 로그인을 마치고 나면 인가서버는 사용자를 이 Url 로 리디렉션 하게 됩니다 이 주소는 인가서버에 등록이 되어 있어야 합니다 

4. scope (scope=email,profile)
인증된 사용자로 부터 제출받은 권한의 범위를 나타냅니다 이때 클라이언트는 여기에 기입된 정보 이상을 취득할 수 없습니다 

5. clientName (clientName=Spring-Oauth2-Authorizaion-client)
clientId 와 마찬가지로 이 클라이언트의 이름을 나타냅니다 생략이 가능합니다 

6. authorizationGrantType (authorizationGrantType=authorization_code)
권한 부여방식을 나타냅니다 이 방식은 크게 4가지 종류가 있습니다 

첫번째로는 authorization Code Grant Type

두번쨰로는 Implicit Grant Type 

세번쨰로는 Resource Owner Password Credentials Grant Type 

네번쨰로는 Client Credentials Grant Type 

다섯번째로는 Refresh Token Grant Type 

여섯번쨰로는 PKEC – enhanced Authorization Code Grant Type 

이중 제일 많이 쓰이는 방식은 authorization Code Grant Type 이는 현재 spring boot 와 KeyClock 이 연동되어 있는 상태이며 리소스 소유자가 인서버로부터 로그인을 하면 
승인코드 발급 후 이 승인코드 발급 클라이언트는 이 승인코드를 통해 엑세스 코드를 발급받게 되고 클라이언트는 이 액세스 코드를 통해서 사용자 정보를 취득하게 됩니다 
우리가 이 OAuth2 를 하는 동안엔 authorization Code Grant Type , Refresh Token Grant Type , PKEC – enhanced Authorization Code Grant Type 에 대해서 공부를 해볼예정입니다 

7. clientAuthenticationMethod(clientAuthenticationMethod=client_secret_post)
인가서버에 아이디 비밀번호를 전달하는 방식을 뜻합니다 post 방식으로 전달을 하는 것입니다 이 외에서 client_secret_basic 방식과 client_secret_jwt 방식이 있습니다 

8. issuerUri(issuerUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project)
이는 ID 토큰 액세스 토큰을 발급하는 주체를 뜻하며 현재 클라이언트가 사용하는 인증서버의 엔드포인트입니다 

9. authorizationUri(authorizationUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth)
이는 클라이언트가 인가서버로 부터 권한 부여를 요청하기 위해 사용되는 엔드포인트입니다 

10. tokenUri(tokenUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token)
9번에서 인증이 완료 되었을때 클라이언트는 9번의 결과로 발급된 승인코드를 tokenUri 요청을 해서 액세스 토큰 또는 리프레시 토큰을 발급받을때 사용하는 엔드포인트입니다 

11. userInfoUri(userInfoUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo)
10번 에서 발급된 액세스 토콘을 이용하여 클라이언트는 리소스 서버에 사용자 정보를 요청합니다 이때 사용되는 엔드포인트입니다 

12. jwkSetUri(jwkSetUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/certs)
이 엔드포인트는 보통 Resource 서버를 구축할때 사용하는것으로 서명검증 및 토큰디코딩할때 사용하는 엔드포인트입니다 

13. userNameAttribute(userNameAttribute=preferred_username)
이는 리소스 서버로 부터 받은 사용자 정보를 해당 식별자를 통해서 가져오게 됩니다 사용자 정보를 불러올때 사용하는 key 값 

많은 내용이긴 합니다만 이 내용을 잘 알고 있어야 다음시간부터 로그인이 되는 과정을 하나하나 살펴볼때 도움이 되며 그때 또한번 설명을 할 예정입니다 





