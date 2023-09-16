---

title: Spring Secuirty 15 JWT 3 JWT 발급과 검증과정 1
author: kimdongy1000
date: 2023-06-13 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

우리는 지난시간에 JWT 발급과 검증이 되는 코드를 보았는데 구체적으로 어떻게 발급이 되고 어떻게 검증이 되는지 알아보겠습니다 

## JWT 발급 소스 코드 

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


## DefaultJwtBuilder 

```
public class DefaultJwtBuilder implements JwtBuilder 



public String compact() {

    if (this.serializer == null) {
        // try to find one based on the services available
        // TODO: This util class will throw a UnavailableImplementationException here to retain behavior of previous version, remove in v1.0
        // use the previous commented out line instead
        this.serializer = LegacyServices.loadFirst(Serializer.class);
    }

    if (payload == null && Collections.isEmpty(claims)) {
        payload = "";
    }

    if (payload != null && !Collections.isEmpty(claims)) {
        throw new IllegalStateException("Both 'payload' and 'claims' cannot both be specified. Choose either one.");
    }

    Header header = ensureHeader();

    JwsHeader jwsHeader;
    if (header instanceof JwsHeader) {
        jwsHeader = (JwsHeader) header;
    } else {
        //noinspection unchecked
        jwsHeader = new DefaultJwsHeader(header);
    }

    if (key != null) {
        jwsHeader.setAlgorithm(algorithm.getValue());
    } else {
        //no signature - plaintext JWT:
        jwsHeader.setAlgorithm(SignatureAlgorithm.NONE.getValue());
    }

    if (compressionCodec != null) {
        jwsHeader.setCompressionAlgorithm(compressionCodec.getAlgorithmName());
    }

    String base64UrlEncodedHeader = base64UrlEncode(jwsHeader, "Unable to serialize header to json.");

    byte[] bytes;
    try {
        bytes = this.payload != null ? payload.getBytes(Strings.UTF_8) : toJson(claims);
    } catch (SerializationException e) {
        throw new IllegalArgumentException("Unable to serialize claims object to json: " + e.getMessage(), e);
    }

    if (compressionCodec != null) {
        bytes = compressionCodec.compress(bytes);
    }

    String base64UrlEncodedBody = base64UrlEncoder.encode(bytes);

    String jwt = base64UrlEncodedHeader + JwtParser.SEPARATOR_CHAR + base64UrlEncodedBody;

    if (key != null) { //jwt must be signed:

        JwtSigner signer = createSigner(algorithm, key);

        String base64UrlSignature = signer.sign(jwt);

        jwt += JwtParser.SEPARATOR_CHAR + base64UrlSignature;
    } else {
        // no signature (plaintext), but must terminate w/ a period, see
        // https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-25#section-6.1
        jwt += JwtParser.SEPARATOR_CHAR;
    }

    return jwt;
}

```

JwtBuilder 를 구현하는 DefaultJwtBuilder 가 Jwt 생성을 담당하게 됩니다 그 안에 있는 compact 를 호출하게 됩니다 이 함수의 끝은 jwt 를 반환하는 것으로 타입은 보시다 싶히 String 입니다 

```

if (payload == null && Collections.isEmpty(claims)) {
    payload = "";
}

if (payload != null && !Collections.isEmpty(claims)) {
    throw new IllegalStateException("Both 'payload' and 'claims' cannot both be specified. Choose either one.");
}

```

이 두줄에서 볼 수 있는것은 claims 가 핵심인데 일단 payload 는 null 이 들어오게 됩니다 

이 Claims 는 우리가 어제 보았듯이 payload 의 핵심 내용을 전달할때 사용하는 key 로 보시면됩니다 


## Claims
```

//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package io.jsonwebtoken;

import java.util.Date;
import java.util.Map;

public interface Claims extends Map<String, Object>, ClaimsMutator<Claims> {
    String ISSUER = "iss";
    String SUBJECT = "sub";
    String AUDIENCE = "aud";
    String EXPIRATION = "exp";
    String NOT_BEFORE = "nbf";
    String ISSUED_AT = "iat";
    String ID = "jti";

    String getIssuer();

    Claims setIssuer(String var1);

    String getSubject();

    Claims setSubject(String var1);

    String getAudience();

    Claims setAudience(String var1);

    Date getExpiration();

    Claims setExpiration(Date var1);

    Date getNotBefore();

    Claims setNotBefore(Date var1);

    Date getIssuedAt();

    Claims setIssuedAt(Date var1);

    String getId();

    Claims setId(String var1);

    <T> T get(String var1, Class<T> var2);
}


```
Claims 는 이와 같이 인터페이스로 구현이 되어 있고 구현체가 아래 있는 get , set 를 구현하게 됩니다 그리고 다시 소스로 돌아와서 



## jwt Header 생성
```

Header header = ensureHeader();

```
여기서 사용하는 헤더는 우리가 흔히 알고 있는  HttpHeader 이 아닌 `public interface Header<T extends Header<T>> extends Map<String, Object> `
마찬가지로 inserface Header 로 Map 을 상속을 받는것을 볼 수 있습니다 이 인터페이스의 구현체는 

## DefaultHeader

```

public class DefaultHeader<T extends Header<T>> extends JwtMap implements Header<T> 

```

가 구현을 하고 있으며 `ensureHeader()` 를 호출할때 

```

protected Header ensureHeader() {
    if (this.header == null) {
        this.header = new DefaultHeader();
    }
    return this.header;
}

```

객체를 만들고 이 DefaultHeader 를 return 하게 됩니다 


```

if (key != null) {
    jwsHeader.setAlgorithm(algorithm.getValue());
} else {
    //no signature - plaintext JWT:
    jwsHeader.setAlgorithm(SignatureAlgorithm.NONE.getValue());
}

```
그리고 key 가 있는지 없는지 보게 되는데 이때 key 는 우리가 서명을 할때 사용하는 개인 key 이 key 가 없으면 서명알고리즘을 선택하지 않게 되고 key 가 있으면 우리가 앞에서 
선택한 암호화 알고리즘을 jwsHeader에 넣게 됩니다 이때 jwsHeader에 는 우리가 위에서 ensureHeader 를 호출하고 DefaultHeader 가 삽입된 객체입니다 (이부분은 생략)

` String base64UrlEncodedHeader = base64UrlEncode(jwsHeader, "Unable to serialize header to json.");`
그리고 이 헤더를 BASE64인코더로 인코딩한 String 을 base64UrlEncodedHeader 저장하게 됩니다 이 결과는 `eyJhbGciOiJIUzUxMiJ9` 가 됩니다 


## Payload 만들기 
```
byte[] bytes;
try {
    bytes = this.payload != null ? payload.getBytes(Strings.UTF_8) : toJson(claims);
} catch (SerializationException e) {
    throw new IllegalArgumentException("Unable to serialize claims object to json: " + e.getMessage(), e);
}

if (compressionCodec != null) {
    bytes = compressionCodec.compress(bytes);
}

String base64UrlEncodedBody = base64UrlEncoder.encode(bytes);
```


이제 payload 를 BASE64로 인코딩하는 모습입니다 안에 있는 claims 정보를 bytes 에 담고 그 정보를 base64UrlEncoder 로 마찬가지로 인코딩 하게 됩니다 
`eyJzdWIiOiJ1c2VyIiwianRpIjoiVGltZSIsImlhdCI6MTY5NDMxMjg2NCwiZXhwIjoxNjk0MzEyOTAwfQ` 그리고 이 정보는 여기에 담기게 됩니다 

그리고 이 jwt는 header 과 body 가 연결이 되는데 

String jwt = base64UrlEncodedHeader + JwtParser.SEPARATOR_CHAR + base64UrlEncodedBody; 이렇게 한번 연결되는데 이때  JwtParser.SEPARATOR_CHAR
는 . 을 의미합니다 jwt 는 각 경계마다 . 으로 연결을 해서 구분하게는 header , payload , sign 에 각각 . 으로 구분을 하게 됩니다

## JWT 서명 
```

if (key != null) { //jwt must be signed:

    JwtSigner signer = createSigner(algorithm, key);

    String base64UrlSignature = signer.sign(jwt);

    jwt += JwtParser.SEPARATOR_CHAR + base64UrlSignature;
} else {
    // no signature (plaintext), but must terminate w/ a period, see
    // https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-25#section-6.1
    jwt += JwtParser.SEPARATOR_CHAR;
}

```

그리고 제일 마지막으로 개인서명을 기존의 jwt 와 결합을 해야 합니다 이떄 key 를 권장하고 있지만 key 가 없어도 만들 수는 있습니다 대신 보안에는 문제가 있게 됩니다 
JwtSigner 가 개인 key 와 알고리즘 방식으로 서명을 만들게 되고 

`String base64UrlSignature = signer.sign(jwt);` 함수를 호출해서 해당 jwt 에 서명을 하는 과정을 거치게 됩니다 이게 이제 제일 마지막에 찍히는 jwt 의 서명 
그리고 만들어진 서명을 기존의 jwt 와 마찬가지로 . 으로 찍고 마무리 하게 됩니다 

그렇게 하나의 jwt 가 만들어지게 됩니다 

우리는 이렇게 jwt 를 하나로 만들어보게되었습니다 다음시간엔 jwt 를 어떻게 검증을 하는지에 대해서 알아보도록 하겠습니다 