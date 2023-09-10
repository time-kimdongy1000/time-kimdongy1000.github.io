---

title: Spring Secuirty 15 JWT 3 JWT 발급과 검증과정 1
author: kimdongy1000
date: 2023-06-13 10:00
categories: [Spring, Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

우리는 지난시간에 JWT 발급과 검증이 되는 코드를 보았는데 구체적으로 어떻게 발급이 되고 어떻게 검증이 되는지 알아보겠습니다 

## JWT 발급 

```

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


```

JWT 발급은 이와 동일하게 갈것이고 우리는 여기에 디버그를 걸어서 어떻게 생성이 되는지 확인을 해볼것이다 

