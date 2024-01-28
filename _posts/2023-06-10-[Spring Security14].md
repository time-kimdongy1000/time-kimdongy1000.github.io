---
title: Spring Secuirty 14 JWT 1 JWT 의 개요 및 발급
author: kimdongy1000
date: 2023-06-10 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

## JWT 
JWT (Json Web Token) 웹표준으로 정의된 토큰기반의 인증 기법중 하나입니다 이는 서버와 클라이언트간의 정보를 JSON 객체로 만들어서 보다 안전하게 전송을 하는 방법중 하나입니다 

## 사용처
Spring Security 의 기본적인 form 로그인에 대해서 공부가 끝이나면 OAuth2.0 에 대해서 공부를 할것인데 이때 JWT 방식을 OAUth2.0 이 채택해서 사용하고 있습니다 

## maven
```
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
	<groupId>io.jsonwebtoken</groupId>
	<artifactId>jjwt-api</artifactId>
	<version>0.11.5</version>
</dependency>
<dependency>
	<groupId>io.jsonwebtoken</groupId>
	<artifactId>jjwt-impl</artifactId>
	<version>0.11.5</version>
	<scope>runtime</scope>
</dependency>
<dependency>
	<groupId>io.jsonwebtoken</groupId>
	<artifactId>jjwt-jackson</artifactId>
	<version>0.11.5</version>
	<scope>runtime</scope>
</dependency>
```

## JWT 생성 핸들러 만들기 
```

@Controller
@RestController
@RequestMapping("jwt")
public class DemoController {

    @GetMapping("/getJwt")
    public String getJwt(@AuthenticationPrincipal UserDetails userDetails){

        return generateToken(userDetails);
    }

    private String generateToken(UserDetails userDetails){

        Date now = new Date();
        Date expirationDate = new Date(now.getTime() + 3600);

        return Jwts.builder()
		.setSubject(userDetails.getUsername())
		.setIssuedAt(now)
		.setExpiration(expirationDate)
		.signWith(SignatureAlgorithm.HS512, "cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3e")
		.compact();
	}
}

```

우리가 로그인을 해서 http://localhost:8080/jwt/getJwt 주소로 요청을 하게 되면 jwt 가 발급이 되는데 소스 분석 이전에 결과를 먼저 보게 되면

![jwt 발급](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/950121ec-a2b6-432f-8259-74c59358dbd4)

이렇게 String 형태의 의미없는 문자열이 쫘악 나열되어 있는것으로 보입니다 다만 이를 분석할 수 있는 사이트가 있는데 

<https://jwt.io/> 로 오게 되면 해당 jwt 를 분석할 수 있습니다 그래서 넣고 보면 이와 같은 형태가 만들어집니다

## 발급된 JWT 디코딩
```
HEADER:ALGORITHM & TOKEN TYPE
{
  "alg": "HS512"
}

PAYLOAD:DATA
{
  "sub": "user",
  "iat": 1694260606,
  "exp": 1694260609
}

VERIFY SIGNATURE
HMACSHA512(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  
your-256-bit-secret

) secret base64 encoded

```

JWT 는 크게 크게 3가지 부분으로 나눠져 있습니다 헤더(Header) , 페이로드(payload) , 서명(SIGNATURE) 가 각각 BASE64 인코딩을 기반으로 구성되어 있습니다

## 헤더(Header)
```
{
  "alg": "HS512"
}

```
헤더는 JWT의 타입 및 사용하는 해시알고리즘 , 메타데이터를 포함하고 있습니다 
typ(타입) 	  : 토큰의 유형을 나타내는데 현재는 생략되어 있지만 대부분 JWT 라고 지정이 됩니다 
alg(알고리즘) : 토큰이 현재 어떤방식으로 암호화 되어 있는지 나타내게 됩니다 이 알고리즘은 뒤에 Decode 할때 사용됩니다 


## 페이로드(payload)

{
  "sub": "user",
  "iat": 1694260606,
  "exp": 1694260609
}

페이로드는 실제로 서버에 보낼 데이터를 말합니다 JWT 에서 이를 클레임이라고 표현을 하며 이들은 KEY-VALUE 형태로 구성이 되어 있습니다 
지금보이는 sub , iat , exp 는 JWT 의 표준클레임 (반드시 들어가야하는 KEY) 그리고 사용자가 임의로 넣을 수 있는 비표준 클레임으로 구성이 되어 있습니다 

표준 클레임은 다음과 같습니다 

sub 토큰의 주체를 나타내는데 이는 주로 사용자를 식별할때 사용하게 됩니다 
iat 토큰이 발급되는 시간을 나타냅니다 
exp 토큰이 만료되는 시간을 나탑니다 

여기 클레임에는 보이지 않지만 몇가지 표준 클레임이 더 존재합니다 

iss 토큰을 발행한 식별자를 나타냅니다 
aud 토큰 발급의 대상자 (수신자) 나타냅니다

## 서명(SIGNATURE)

HMACSHA512(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  
your-256-bit-secret

) 
서명은 위의 헤더부분과 , 페이로드 부분을 합쳐서 생성되며 이 서명의 역활은 토큰이 변조되지 않았음을 보증하는 역활을 하게 됩니다 서명은 JWT.io 페이지 같은데서 볼 수 없고 
서버측에서는 주어진 데이터 (헤더 , 페이로드)를 기반으로 만들어지게 됩니다 이때 헤더의 알고리즘으로 암호화가 진행이 됩니다 

## 서버에서 서명 검증 과정 
1. 전달된 JWT 를 .(점) 기준으로 헤더와 , 페이로드와 , 서명을 분리합니다 

2. 헤더와 페이로드를 각각 BASE64 디코딩 및 JSON 객체로 만듭니다 

3. 헤더에 있는 암호화 알고리즘을 추출합니다 이때 암호화 알고리즘이 대칭키/비대칭키인지 분간합니다

4. 대칭키면 서버에 저장된 시크릿키를 가지고 다시 한번 헤더와 , 페이로드를 조합해서 서명을 만듭니다 이때 클라이언트에서 전달된 서명과 일치하면 이 JWT 는 변조 되지 않았습니다 
   비대칭키면 클라이언트에서 전달된 KEY 는 비밀키로 서명이 된것이기 때문에 서버는 이에 해당하는 공개키로 서명을 만들게 됩니다 마찬가지로 전달된 서명과 일치하면 이 JWT 는 변조되지 않았음을 보증합니다 

5. 만약 서명하고 다른 부분이 발견되면 이는 유효하지 않은것을 판단해 폐기 시키고 클라이언트에게는 401 에러를 return 시킵니다 



그럼 아래 예제에서 개인key 를 가지고 간단한 JWT 를 만드는 과정을 한번 보겠습니다 

## jwt 생성기 
```
private String generateToken(UserDetails userDetails){

	Date now = new Date();
	Date expirationDate = new Date(now.getTime() + 3600);

	return Jwts.builder()
		.setSubject(userDetails.getUsername())
		.setIssuedAt(now)
		.setExpiration(expirationDate)
		.signWith(SignatureAlgorithm.HS512, "cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3e")
		.compact();
}

```

Jwts 를 통해서 jwt 를 생성하며 setSubject 를 통해서 sub 를 setIssuedAt 발급 시간을 setExpiration 만료시간을 signWith 를 통해서 인코딩 알고리즘과 , 개인 key 를 이용해서 jwt 를 발급한 모습입니다 우리는 앞으로 이 jwt 를 활용해서 인증서버와 리소스 서버를 따로 구축할것입니다