---
title: Spring Secuirty 43 Refresh_Token 
author: kimdongy1000
date: 2023-07-17 16:30
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

## 소스 전체
<https://gitlab.com/kimdongy1000/public_project_amadeus/-/tree/main?ref_type=heads>

해당 소스는 민감한 정보를 제외한 순수 코드입니다 사용하실려면 application.yml 에 자신이 필요한 정보를 기입하시면 사용 가능합니다 
해당 글을 적는부분과 소스의 올라간 부분은 상당히 많이 다릅니다 

오늘은 코드 대신에 다음에 할 Refresh_Token 에 대해서 잠깐 이론적인 부분을 보고 진행을 하겠습니다

우린 지난시간에 naver 로그인을 이용해서 google 처럼 마찬가지로 access_token 을 발급받고 그 내용을 바탕으로 유저 정보를 완료하여 로그인한 사용자의 정보를 받아서 그 정보를 바탕으로 다시 우리 어플리케이션 방식으로 JWT 토큰을 생성해서 다시 return 을 해주었습니다 그러면서 어렴풋이 Refresh_Token 에 대해서 잠깐 코드만 보고 지나갔습니다 그 부분을 다시보면

## Refresh_Token 만드는 소스 일부분
```
/*refresh_token*/

JWSHeader refresh_JWTHeader = new JWSHeader.Builder(jwsAlgorithm).keyID(keyId).build();
JWTClaimsSet refresh_jwtPayload = new JWTClaimsSet.Builder().subject("refresh_token_user")
                                        .issuer(backend_server)
                                        .claim("username" , authentication.getPrincipal().toString() + "/" + UUID.randomUUID().toString() + "/" + new Date().toString())
                                        .expirationTime(new Date(new Date().getTime() + (refresh_tokenExpireTime)))
                                        .build();

SignedJWT refresh_signedJWT = new SignedJWT(refresh_JWTHeader , refresh_jwtPayload);
refresh_signedJWT.sign(jwsSigner);

String refresh_token = refresh_signedJWT.serialize();


Map<String , Object> result_token = new HashMap<>();
result_token.put("ACCESS_TOKEN"     , access_token);
result_token.put("REFRESH_TOKEN"    , refresh_token);


ValueOperations<String , String> jwt_token_key_value = redisTemplate.opsForValue();
String refresh_redis_key = refresh_token;
String refresh_redis_value = gson.toJson(authentication);
Duration refresh_token_expireTime = Duration.ofMillis(refresh_tokenExpireTime);

```

이 부분을 보되 오늘은 코드 보다도 Refresh_Token 에 대해서 집중을 해보겠습니다 

우리가 현재 사용하는 토큰의 종류는 총 2가지 access_token , refresh_token 이 있습니다 

## access_token
access_token 은 사용자의 신원을 파악하기 위한 토큰입니다 우리가 요청에서 해당 토큰을 헤더에 첨부해서 보내면 이 토큰은 개인키 또는 공개키로 복호화 해서 위변조 및 서명 및 시간을 체크한 후 올바르게 발급이 되었다면 SecurityContext 에 setAuthentication 하고 인증을 마치게 됩니다 그렇기 때문에 사실 이 access_token 같은 경우는 신원을 확인하기 위한 가장 중요한 토큰입니다 자칫 잘못해서 이 토큰이 노출이 되어버리면 신원이 불명확한 사람이 웹사이트에 접근을 해서 마치 그 사람인양 행세를 할 수 있기 때문입니다 그렇기에 보통 access_token 은 만료시간(expireTime) 을 비교적 짧게 둡니다 이 만료시간이 지나게 되면 사용자는 다시금 로그인을 해서 access_token 을 발급을 받아야 합니다

## refresh_token
access_token 의 짧은 만료시간에 대비해서 나온 토큰입니다 access_token 의 주기가 짧으면 짧을 수록 사용자는 access_toen 이 만료될때 마다 다시금 인증/인가를 받기 위해 웹사이트에 재로그인을 해야 합니다 그러면 사용자 경험은 매우 떨어지게 됩니다 그렇기에 만약 access_token 이 만료가 되었을때 refresh_token 을 둬서 이 토큰이 유효한다면 로그인 절차 없이 다시 access_token 을 반환하면서 사용자의 계속적인 (지속적인) 웹사이트 경험을 하는것이 목적입니다 그런데 문제가 있습니다 access_token 은 주기가 짧아서 탈취 당하더라도
짧은 시간내에 만료가 됨으로 크게 문제가 없지만 refresh_token 은 같은 경우는 짧아야 한달 , 길면 1년  , 무제한 으로 지속이 되는 토큰의 특정상 탈취가 되면 보안에 문제가 있을 수 있습니다 그렇기에 보통 refresh_token 으로 access_token 을 요청할시 기존 refresh_token 만료 시키는 방법을 따르게 됩니다 그러나 이 방식을 하게 되면 JWT 의 고유한  SERVER STAYLESS 방식을 부정하는 것이 됩니다 

## SERVER STAYLESS
우리는 다음장부터 Refresh_token 을 redis 라는 서드파티 DB 에 Refesh_token 을 저장을 할것입니다 그렇게 되면 토큰의 장점중 하나인 서버가 사용자의 상태를 확인할 이유가 없었지만 다시금 
DB에 저장을 함으로서  SERVER STAYLESS 이 다시 SERVER SAY 방식으로 바뀌게 됩니다 보통 JWT 를 사용하는 곳에서는 사용자의 상태를 서버가 알 수 없게 하는 것이 목적인데 refresh_token 을 서버에 저장을 함으로서 다시금 서버에서 해당 사용자의 상태를 체크하고 있는게 아닌가 하는 의문이 있을 수 있습니다 그럼에도 불구하고 이렇게 하는 이유는 결국은 보안때문입니다 
잘생각해보면 이런 어플리케션을 본적이 있을 것입니다 휴대폰 어플리케이션에서 내가 로그인한 모든곳 로그아웃하기 기능을 한번쯤 사용해보셨을것입니다 그럼 이런 기능들은 구현하기 위해서 사실상 서버에 사용자의 정보가 저장이 되어 있어야만 해당 토큰을 원격으로 만료시키고 자동 로그인을 설정해둔 모든 어플리케이션에서 로그아웃 할 수 있게 만든 기능입니다 
결국 사용자의 경험과 보안을 적절히 섞었을때는 다시 서버는 사용자의 상태를 알고 있어야 한다는 것입니다 

오늘은 토큰의 종류와 SERVER STAYLESS 알아보았고 다음 시간에는 redis 와 연동하는 부분에 대해서 글을 적어나가겠습니다 




