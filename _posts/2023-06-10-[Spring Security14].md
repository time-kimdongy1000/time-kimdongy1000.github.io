---
title: Spring Secuirty 14 JWT 1 JWT 의 개요 및 발급
author: kimdongy1000
date: 2023-06-10 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

우리는 지난시간까지 회원가입 페이지를 직접 만들고 커스텀한 로그인 방식에 대해서도 공부를 했다 이번시간에는 다음에 할 미니 프로젝트를 위해서 JWT 에 대한것을 먼저 배우고 시작을 하겠습니다 

JWT 는 JSON WEB TOKEN  먼저 토큰에 대해서 알아보자 토큰은 애플리케이션이 사용자를 인증했음을 증명하는 방법을 제공해 사용자가 애플리케이션 리소스에 엑세스 할 수 있게 한다 쉽게 말해서 우리는 회사 사원증이라고 생각하면된다 사원증이 있는 사람은 회사가 설정한 권한에 따라서 접근할 수 있는 층이 존재하는 반면 접근 할 수 없는 층이 존재하기도 한다 우리가 JWT 를 배우느게 되는 이유는 어플리케이션이 크면 클수록 인증서버와 , 리소스 서버를 분리하는 경우가 많게 되는데 그때 가장 많이 이용하는 방식으로 JWT 토큰 방식이 제일 많이 이용이 된다 

## JSON Web Token 
앞에 Json 이 붙은 이유는 이 토큰의 형식이 JSON 형태의 key - value 형태의 토큰이라는 뜻입니다 이 토큰은 특별한 알고리즘으로 인코딩되며 서명을 통해 이 토큰이 어디서 발행이 되었는지 알 수 있습니다 일단 간단하게 jwt 를 발급하는 예제를 만들고 한번 분석을 해보겠습니다 

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


</dependencies>
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
JWT 는 크게 헤더 , 페이로드 , 서명 3개의 부분으로 나눠지게 됩니다


## HEADER
```
{
  "alg": "HS512"
}

```
헤더는 크게 알고리즘과 타입을 포함하게 됩니다 
typ(타입) 	  : 토큰의 유형을 나타내는데 현재는 생략되어 있지만 대부분 JWT 라고 지정이 됩니다 
alg(알고리즘) : 토큰이 현재 어떤방식으로 암호화 되어 있는지 나타내게 됩니다 이 알고리즘은 뒤에 Decode 할때 사용됩니다 


## PAYLOAD

{
  "sub": "user",
  "iat": 1694260606,
  "exp": 1694260609
}

본문 같은 경우는 특정한 정보를 전달할려고 하는 내용자체가 담겨 있습니다 표준적으로 전달하고자 하는 내용이 key 로 정의가 되어 있고 발급자는 그에 맞춰서 토큰을 발급을 해야합니다 지금보이는 sub , iat , exp 는 표준클레임이며 각각 그 내용과 뜻이 담겨 있습니다 

sub 토큰의 주체를 나타내는데 이는 주로 사용자를 식별할때 사용하게 됩니다 
iat 토큰이 발급되는 시간을 나타냅니다 
exp 토큰이 만료되는 시간을 나탑니다 

여기 클레임에는 보이지 않지만 몇가지더 표준 클레임이 정의되어 있습니다 

iss 토큰을 발행한 식별자를 나타냅니다 
aud 토큰 발급의 대상자 (수신자) 나타냅니다

## 서명 SIGNATURE

HMACSHA512(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  
your-256-bit-secret

) 

서명은 현재 드러나지 않게 됩니다 그 이유는 토큰의 무결성 검증과 관련이 있습니다 이 서명은 토큰의 발급자가 토큰이 변조 되지 않았음을 보증하며 

서버에서 생성여부를 확인할 수 있습니다 서버에서 개인키를 통해서 jwt 가 생성이 되고 또 서버로 전달된 jwt 를 검증하는 과정에서 이 개인 key 를 활용해서 무결성을 검증하게 됩니다 다만 이 개인 key 가 탈취 되었을 경우 토큰의 무결성을 보장할 수 없기에 보안에 힘을 쓰거나 토큰을 주기적으로 변경을 해야 합니다 
(지금은 개인key 로 서명하고 개인key 로 복호화 할 예정입니다 Oauth2 에서는 개인key 로 암호화 하고 공개key 로 복호화 합니다)


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