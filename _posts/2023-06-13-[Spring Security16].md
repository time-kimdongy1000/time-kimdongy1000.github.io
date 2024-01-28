---
title: Spring Secuirty 16 JWT 3 JWT 발급과 검증과정 1
author: kimdongy1000
date: 2023-06-13 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

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
        jwsHeader = new DefaultJwsHeader(header);
    }

    if (key != null) {
        jwsHeader.setAlgorithm(algorithm.getValue());
    } else {
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

    if (key != null) { 

        JwtSigner signer = createSigner(algorithm, key);

        String base64UrlSignature = signer.sign(jwt);

        jwt += JwtParser.SEPARATOR_CHAR + base64UrlSignature;
    } else {
        jwt += JwtParser.SEPARATOR_CHAR;
    }
    return jwt;
}

```
DefaultJwtBuilder jjwt 라이브러리에서 제공되는 클래스로 JWT를 생성하는 아주 기본적인 JWT 생성기입니다 

```
if (payload == null && Collections.isEmpty(claims)) {
    payload = "";
}

if (payload != null && !Collections.isEmpty(claims)) {
    throw new IllegalStateException("Both 'payload' and 'claims' cannot both be specified. Choose either one.");
}
```
payload 가 null 이거나 claims 가 비어 있다면 payload= "" 로 초기화를 진행을 하는데 만약 payload 가 null 이 아니면서 claims 가 비어있지 않으면 
Both 'payload' and 'claims' cannot both be specified. Choose either one. payload 와 claims 를 둘다 사용할 수 없습니다 둘중 하나만 선택을 해서 써야 합니다 

## payload 를 쓰는 예제
```
Jwts.builder()
    .setSubject("user123")
    .claim("name", "John Doe")
    .claim("admin", true)
    .signWith(secretKey)
    .compact();
```

## Claims  를 쓰는 예제
```
Claims claims = Jwts.claims()
    .setSubject("user123")
    .setAudience("audience123");

Jwts.builder()
    .setClaims(claims)
    .signWith(secretKey)
    .compact();
```
즉 이 둘을 동시에 사용하지 말아야 한다는 뜻입니다 

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
가 구현을 하고 있으며 `ensureHeader()` 를 호출할때 DefaultHeader 를 return 하게 됩니다 

```
protected Header ensureHeader() {
    if (this.header == null) {
        this.header = new DefaultHeader();
    }
    return this.header;
}

```

## 헤더에 암호화 알고리즘 세팅 
```
if (key != null) {
    jwsHeader.setAlgorithm(algorithm.getValue());
} else {
    //no signature - plaintext JWT:
    jwsHeader.setAlgorithm(SignatureAlgorithm.NONE.getValue());
}

```
이때 암호화 알고리즘을 넣게 되는데 이는 우리가 위에서 Builder 를 만들때 사용한 `.signWith(SignatureAlgorithm.HS512, SECRET_KEY)` 이 부분을 사용하게 됩니다 
그리고 만약 이때 선택한 알고리즘이 없으면 이때는 알고리즘을 선택하지는 않지만 마찬가지로 서명또한 없게 됩니다 

`String base64UrlEncodedHeader = base64UrlEncode(jwsHeader, "Unable to serialize header to json.");`
그리고 위에서 만들어진 헤더를 base64로 인코딩해서 인코딩된 헤더를 만들게됩니다 


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
그리고 이렇게 만들어진 정보는 앞의 헤더 부분과 .(점) 으로 구분되어서 헤더와 페이로드가 만들어지게 됩니다 
`String jwt = base64UrlEncodedHeader + JwtParser.SEPARATOR_CHAR + base64UrlEncodedBody; ` 이때 ` JwtParser.SEPARATOR_CHAR` 는 .(점) 을 뜻합니다 

## JWT 서명 
```
if (key != null) { 

    JwtSigner signer = createSigner(algorithm, key);

    String base64UrlSignature = signer.sign(jwt);

    jwt += JwtParser.SEPARATOR_CHAR + base64UrlSignature;

} else {

    jwt += JwtParser.SEPARATOR_CHAR;
}
```
이제 마지막으로 서명부분을 만들게 되는데 sign는 앞에서 만들어진 헤더와 , 페이로드를 통해서 만들어지게 됩니다 이때 암호화 알고리즘을 사용해서 만들어지고 이를 마찬가지로 . 을 기준으로 
BASE64로 인코딩을 해서 페이로드 다음으로 붙여서 반환하게 됩니다 