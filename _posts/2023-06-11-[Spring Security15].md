---
title: Spring Secuirty 15 JWT 2 JWT 유효성 검증
author: kimdongy1000
date: 2023-06-11 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

우리는 지난시간에 JWT 의 개요 및 생성을 했습니다 이번시간에는 발급된 JWT 를 검증한은 로직을 만들어보겠습니다 이번시간에는 POST - MAN 을 활용해서 
간단한 서버 요청을 만들어보겠습니다

## JWT 디코딩

```
@Controller
@RestController
@RequestMapping("jwt")
public class DemoController {

    private static final String SECRET_KEY = "cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3p";



    @GetMapping("/getJwt")
    public String getJwt(@AuthenticationPrincipal UserDetails userDetails){



        return generateToken(userDetails);
    }

    @GetMapping("/validJwt")
    public String validJwt(@RequestParam("jwt") String jwt){

        return parseJwt(jwt) + "";
    }

    private String generateToken(UserDetails userDetails){

        Date now = new Date();
        Date expirationDate = new Date(now.getTime() + 36000);

        return Jwts.builder()
                .setSubject(userDetails.getUsername())
                .setId("Time")
                .setIssuedAt(now)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS512, SECRET_KEY)
                .compact();

    }

    private boolean parseJwt(String jwt){

        boolean returnFlag = false;
        try{


            Claims claims = Jwts.parserBuilder()
                    .setSigningKey(SECRET_KEY)
                    .build()
                    .parseClaimsJws(jwt)
                    .getBody();

            returnFlag = true;

        }catch(ExpiredJwtException e1){

            System.out.println(e1.getMessage());
            System.out.println("jwt 시간만료");
            returnFlag = false;

        }catch(MalformedJwtException e2){

            System.out.println(e2.getMessage());
            System.out.println("jwt 형식이 아닙니다");
            returnFlag = false;

        }catch (UnsupportedJwtException e3){

            System.out.println(e3.getMessage());
            System.out.println("지원되지 않는 클레임입니다 ");
            returnFlag = false;

        }catch (SignatureException e4){
            System.out.println(e4.getMessage());
            System.out.println("개인키가 틀립니다 ");
            return false;
        }

        return returnFlag;
    }
}
```
비밀key 는 상단에 정의를 해두고 하단에 새로운 메서드로 parseJwt 를 만들겠습니다 


```
Claims claims = Jwts.parserBuilder()
	.setSigningKey(SECRET_KEY)
	.build()
	.parseClaimsJws( )
	.getBody();

```
 
claims 로 parserBuilder SECRET_KEY jwt 를 넣으면 기본적으로 jwt 형식 체크 , 개인키 검증 , 만료시간 검증 등등 진행하게 됩니다 만약 jwt 에 대해서 위와 같은 불일치 사유가 나오면에러를 던지고 끝나게 됩니다 그래서 저는 try - catch 로 아래와 같은 에러 로직을 만들고 테스트를 진행을 했습니다 


```
JWT signature does not match locally computed signature. JWT validity cannot be asserted and should not be trusted.
개인키가 틀립니다 
JWT expired at 2023-09-09T13:16:05Z. Current time: 2023-09-09T13:18:13Z, a difference of 128631 milliseconds.  Allowed clock skew: 0 milliseconds.
jwt 시간만료
```