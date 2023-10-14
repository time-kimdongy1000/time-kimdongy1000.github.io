---

title: Spring Secuirty 28 OIDC 란 무엇인가?
author: kimdongy1000
date: 2023-07-05 14:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

먼저 OIDC 가 설명하기 전에 OAuth2 의 계층에 대해서 한번 알아볼려고 합니다 





## 인가서버 프로토콜 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/e3def2ee-4efd-4b98-891f-02976dacab74)

우리는 이제까지 제일 하위계층인 OAuth2.0 에 대해서 공부를 진행을 했습니다 그리고 이 상위계층에는 OpendID Connect 라는 계층이 존재합니다 
우리는 이를 줄여서 OIDC 라고 할것이다 

## OIDC 란

OIDC는 "OpenID Connect"의 약자로, 웹 및 모바일 애플리케이션에서 사용자 인증 및 인가를 위한 표준 프로토콜 및 인증 계층입니다. 
OIDC는 웹 응용 프로그램 및 서비스에 통합 사용자 인증 인가 및 권한을 부여하기 위한 프레임워크 

## OAuth2 와 OIDC 의 차이점 
사실 OIDC 의 설명만 보면 OAuth2 와 하는게 다를 봐가 없다고 생각이 든다 하지만 이 둘 인증/인가와 권한 부여면에서는 동일하지만 
사용처를 보면 조금 다른면이 있다 

## OAuth2 목적
애플리케이션이 제한된 권한으로 제 3의 어플리케이션에 접근하도록 권한을 부여하는 프레임워크 입니다 
예를 들어서 OAuth2 는 애플리케이션이 구글 캘린더에 접근해서 데이터를 가져올려고 할때 구글 계정으로 로그인을 하도록 돕는 프레임워크 

## OIDC 목적
사용자의 신원확인에 중점을 둔 인증 프로토콜입니다 
주로 SSO 에서 사용됩니다 SSO 란 Single Sign-On 으로 하나의 로그인으로 여타 다른 사이트 또는 응용프로그램에 대한 단일 인증을 제공하는 프레임워크인데 
OIDC 는 주로 이곳에서 많이 쓰이게 됩니다 

그래서 이 둘은 토큰을 반환하기는 하지만 토큰의 내용이 조금씩 다릅니다 

## OAuth2 의 토큰
OAuth2 는 주로 리소스 서버에 대한 권한을 나타내는데 중점으로 만들어진 토큰입니다 

## OIDC 의 토큰 
사용자의 ID와 사용자 정보를 포함하며 사용자의 신원을 확인하는데 사용됩니다

그럼 우리는 앞에서 User 정보를 들고온건 무엇인가요 할 수 있습니다 이는 access_token 으로 사용자 정보를 요청을 해서 사용자 정보를 가져오는 경우지 
실세로 access_token 에는 사용자 정보가 담겨 있지 않습니다 


## OAuaht2 의 토큰 

```

{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "dp7lpFYPY-do8i9U6Vp3sqb4atyutsw1DUQtZZiwI_s"
}


{
  "exp": 1697256009,
  "iat": 1697255709,
  "auth_time": 1697254345,
  "jti": "a66154cb-cca2-417e-9e47-1a87e3fd9bd8",
  "iss": "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project",
  "aud": "account",
  "sub": "30283228-fa36-4d88-82c3-c9494dc44bfd",
  "typ": "Bearer",
  "azp": "Spring-Oauth2-Authorizaion-client",
  "session_state": "0e56893d-0d86-46af-8d72-f185e7ecf483",
  "acr": "0",
  "realm_access": {
    "roles": [
      "offline_access",
      "uma_authorization",
      "default-roles-srping-oauth2-authorizaion-project"
    ]
  },
  "resource_access": {
    "account": {
      "roles": [
        "manage-account",
        "manage-account-links",
        "view-profile"
      ]
    }
  },
  "scope": "email profile",
  "sid": "0e56893d-0d86-46af-8d72-f185e7ecf483",
  "email_verified": false,
  "name": "time user",
  "preferred_username": "user1",
  "given_name": "time",
  "family_name": "user",
  "email": "user1@gmail.com"
}

```

access_token 을 jwt.io 에서 변환을 하면 이렇게 보이게 된다 여기 토큰에는 사용자 정보가 포함이 되지 않는것을 볼 수 있지만 
다음시간부터 할 OIDC 에는 토큰의 승인코드를 보낼시 바로 사용자 정보가 포함된 토큰을 반환하게 됩니다 


다음시간부터 OIDC 에 관련한 이야기를 진행을 하도록 하겠습니다 


