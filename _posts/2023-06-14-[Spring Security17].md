---

title: Spring Secuirty 15 JWT 4 JWT 발급과 검증과정 2
author: kimdongy1000
date: 2023-06-14 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

지난시간에 JWT 발급에대해서 공부했는데 이번에는 이제 발급된 JWT 가 되돌아올때 JWT 를 유효성검사에 대해서 알아보겠습니다

## JWT 검증소스 

```


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

```

여기에 parseClaimsJws 호출을 통해서 jwt 유효성 검사를 하게 됩니다 

## DefaultJwtParse 

```

public class DefaultJwtParser implements JwtParser 

@Override
public Jws<Claims> parseClaimsJws(String claimsJws) {
    return parse(claimsJws, new JwtHandlerAdapter<Jws<Claims>>() {
        @Override
        public Jws<Claims> onClaimsJws(Jws<Claims> jws) {
            return jws;
        }
    });
}

```

이 함수를 호출함과 동시에 jwt 유효성을 검사하게 됩니다 이에 따른 jwt 는 

```

@Override
public <T> T parse(String compact, JwtHandler<T> handler)
    throws ExpiredJwtException, MalformedJwtException, SignatureException {
    Assert.notNull(handler, "JwtHandler argument cannot be null.");
    Assert.hasText(compact, "JWT String argument cannot be null or empty.");

    Jwt jwt = parse(compact);

    if (jwt instanceof Jws) {
        Jws jws = (Jws) jwt;
        Object body = jws.getBody();
        if (body instanceof Claims) {
            return handler.onClaimsJws((Jws<Claims>) jws);
        } else {
            return handler.onPlaintextJws((Jws<String>) jws);
        }
    } else {
        Object body = jwt.getBody();
        if (body instanceof Claims) {
            return handler.onClaimsJwt((Jwt<Header, Claims>) jwt);
        } else {
            return handler.onPlaintextJwt((Jwt<Header, String>) jwt);
        }
    }
}

```
` Jwt jwt = parse(compact);` 함수를 호출을 하게 되는데 이떄 이 parse 가 정말로 깁니다 

## parse 
```

@Override
public Jwt parse(String jwt) throws ExpiredJwtException, MalformedJwtException, SignatureException {

    // TODO, this logic is only need for a now deprecated code path
    // remove this block in v1.0 (the equivalent is already in DefaultJwtParserBuilder)
    if (this.deserializer == null) {
        // try to find one based on the services available
        // TODO: This util class will throw a UnavailableImplementationException here to retain behavior of previous version, remove in v1.0
        this.deserializeJsonWith(LegacyServices.loadFirst(Deserializer.class));
    }

    Assert.hasText(jwt, "JWT String argument cannot be null or empty.");

    if ("..".equals(jwt)) {
        String msg = "JWT string '..' is missing a header.";
        throw new MalformedJwtException(msg);
    }

    String base64UrlEncodedHeader = null;
    String base64UrlEncodedPayload = null;
    String base64UrlEncodedDigest = null;

    int delimiterCount = 0;

    StringBuilder sb = new StringBuilder(128);

    for (char c : jwt.toCharArray()) {

        if (c == SEPARATOR_CHAR) {

            CharSequence tokenSeq = Strings.clean(sb);
            String token = tokenSeq != null ? tokenSeq.toString() : null;

            if (delimiterCount == 0) {
                base64UrlEncodedHeader = token;
            } else if (delimiterCount == 1) {
                base64UrlEncodedPayload = token;
            }

            delimiterCount++;
            sb.setLength(0);
        } else {
            sb.append(c);
        }
    }

    if (delimiterCount != 2) {
        String msg = "JWT strings must contain exactly 2 period characters. Found: " + delimiterCount;
        throw new MalformedJwtException(msg);
    }
    if (sb.length() > 0) {
        base64UrlEncodedDigest = sb.toString();
    }


    // =============== Header =================
    Header header = null;

    CompressionCodec compressionCodec = null;

    if (base64UrlEncodedHeader != null) {
        byte[] bytes = base64UrlDecoder.decode(base64UrlEncodedHeader);
        String origValue = new String(bytes, Strings.UTF_8);
        Map<String, Object> m = (Map<String, Object>) readValue(origValue);

        if (base64UrlEncodedDigest != null) {
            header = new DefaultJwsHeader(m);
        } else {
            header = new DefaultHeader(m);
        }

        compressionCodec = compressionCodecResolver.resolveCompressionCodec(header);
    }

    // =============== Body =================
    String payload = ""; // https://github.com/jwtk/jjwt/pull/540
    if (base64UrlEncodedPayload != null) {
        byte[] bytes = base64UrlDecoder.decode(base64UrlEncodedPayload);
        if (compressionCodec != null) {
            bytes = compressionCodec.decompress(bytes);
        }
        payload = new String(bytes, Strings.UTF_8);
    }

    Claims claims = null;

    if (!payload.isEmpty() && payload.charAt(0) == '{' && payload.charAt(payload.length() - 1) == '}') { //likely to be json, parse it:
        Map<String, Object> claimsMap = (Map<String, Object>) readValue(payload);
        claims = new DefaultClaims(claimsMap);
    }

    // =============== Signature =================
    if (base64UrlEncodedDigest != null) { //it is signed - validate the signature

        JwsHeader jwsHeader = (JwsHeader) header;

        SignatureAlgorithm algorithm = null;

        if (header != null) {
            String alg = jwsHeader.getAlgorithm();
            if (Strings.hasText(alg)) {
                algorithm = SignatureAlgorithm.forName(alg);
            }
        }

        if (algorithm == null || algorithm == SignatureAlgorithm.NONE) {
            //it is plaintext, but it has a signature.  This is invalid:
            String msg = "JWT string has a digest/signature, but the header does not reference a valid signature " +
                "algorithm.";
            throw new MalformedJwtException(msg);
        }

        if (key != null && keyBytes != null) {
            throw new IllegalStateException("A key object and key bytes cannot both be specified. Choose either.");
        } else if ((key != null || keyBytes != null) && signingKeyResolver != null) {
            String object = key != null ? "a key object" : "key bytes";
            throw new IllegalStateException("A signing key resolver and " + object + " cannot both be specified. Choose either.");
        }

        //digitally signed, let's assert the signature:
        Key key = this.key;

        if (key == null) { //fall back to keyBytes

            byte[] keyBytes = this.keyBytes;

            if (Objects.isEmpty(keyBytes) && signingKeyResolver != null) { //use the signingKeyResolver
                if (claims != null) {
                    key = signingKeyResolver.resolveSigningKey(jwsHeader, claims);
                } else {
                    key = signingKeyResolver.resolveSigningKey(jwsHeader, payload);
                }
            }

            if (!Objects.isEmpty(keyBytes)) {

                Assert.isTrue(algorithm.isHmac(),
                    "Key bytes can only be specified for HMAC signatures. Please specify a PublicKey or PrivateKey instance.");

                key = new SecretKeySpec(keyBytes, algorithm.getJcaName());
            }
        }

        Assert.notNull(key, "A signing key must be specified if the specified JWT is digitally signed.");

        //re-create the jwt part without the signature.  This is what needs to be signed for verification:
        String jwtWithoutSignature = base64UrlEncodedHeader + SEPARATOR_CHAR;
        if (base64UrlEncodedPayload != null) {
            jwtWithoutSignature += base64UrlEncodedPayload;
        }

        JwtSignatureValidator validator;
        try {
            algorithm.assertValidVerificationKey(key); //since 0.10.0: https://github.com/jwtk/jjwt/issues/334
            validator = createSignatureValidator(algorithm, key);
        } catch (WeakKeyException e) {
            throw e;
        } catch (InvalidKeyException | IllegalArgumentException e) {
            String algName = algorithm.getValue();
            String msg = "The parsed JWT indicates it was signed with the '" + algName + "' signature " +
                "algorithm, but the provided " + key.getClass().getName() + " key may " +
                "not be used to verify " + algName + " signatures.  Because the specified " +
                "key reflects a specific and expected algorithm, and the JWT does not reflect " +
                "this algorithm, it is likely that the JWT was not expected and therefore should not be " +
                "trusted.  Another possibility is that the parser was provided the incorrect " +
                "signature verification key, but this cannot be assumed for security reasons.";
            throw new UnsupportedJwtException(msg, e);
        }

        if (!validator.isValid(jwtWithoutSignature, base64UrlEncodedDigest)) {
            String msg = "JWT signature does not match locally computed signature. JWT validity cannot be " +
                "asserted and should not be trusted.";
            throw new SignatureException(msg);
        }
    }

    final boolean allowSkew = this.allowedClockSkewMillis > 0;

    //since 0.3:
    if (claims != null) {

        final Date now = this.clock.now();
        long nowTime = now.getTime();

        //https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-30#section-4.1.4
        //token MUST NOT be accepted on or after any specified exp time:
        Date exp = claims.getExpiration();
        if (exp != null) {

            long maxTime = nowTime - this.allowedClockSkewMillis;
            Date max = allowSkew ? new Date(maxTime) : now;
            if (max.after(exp)) {
                String expVal = DateFormats.formatIso8601(exp, false);
                String nowVal = DateFormats.formatIso8601(now, false);

                long differenceMillis = maxTime - exp.getTime();

                String msg = "JWT expired at " + expVal + ". Current time: " + nowVal + ", a difference of " +
                    differenceMillis + " milliseconds.  Allowed clock skew: " +
                    this.allowedClockSkewMillis + " milliseconds.";
                throw new ExpiredJwtException(header, claims, msg);
            }
        }

        //https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-30#section-4.1.5
        //token MUST NOT be accepted before any specified nbf time:
        Date nbf = claims.getNotBefore();
        if (nbf != null) {

            long minTime = nowTime + this.allowedClockSkewMillis;
            Date min = allowSkew ? new Date(minTime) : now;
            if (min.before(nbf)) {
                String nbfVal = DateFormats.formatIso8601(nbf, false);
                String nowVal = DateFormats.formatIso8601(now, false);

                long differenceMillis = nbf.getTime() - minTime;

                String msg = "JWT must not be accepted before " + nbfVal + ". Current time: " + nowVal +
                    ", a difference of " +
                    differenceMillis + " milliseconds.  Allowed clock skew: " +
                    this.allowedClockSkewMillis + " milliseconds.";
                throw new PrematureJwtException(header, claims, msg);
            }
        }

        validateExpectedClaims(header, claims);
    }

    Object body = claims != null ? claims : payload;

    if (base64UrlEncodedDigest != null) {
        return new DefaultJws<>((JwsHeader) header, body, base64UrlEncodedDigest);
    } else {
        return new DefaultJwt<>(header, body);
    }
}

```
이 부분이 JWT 의 유효성 검사를 하게 되는데 너무 긴데 하나씩 가보도록 하겠습니다 

```

Assert.hasText(jwt, "JWT String argument cannot be null or empty.");

if ("..".equals(jwt)) {
    String msg = "JWT string '..' is missing a header.";
    throw new MalformedJwtException(msg);
}

```
먼저 jwt 가 존재하는지 검사하고 jwt 없이 이를 연결하는 .. 만 오게 되는지 찾게 됩니다 그때는 MalformedJwtException 을 던져서 에러를 처리합니다 에러 도 
`String msg = "JWT string '..' is missing a header.";` JWT 헤더를 찾을 수 없어 이렇게 나오게 되네요

```
String base64UrlEncodedHeader = null;
String base64UrlEncodedPayload = null;
String base64UrlEncodedDigest = null;

```

. 을 기준으로 각 header , payload sign 을 구분할 수 있기에 일단 String 으로 구분할 수 있게 각 변수를 선언합니다 

```

 for (char c : jwt.toCharArray()) {

    if (c == SEPARATOR_CHAR) {

        CharSequence tokenSeq = Strings.clean(sb);
        String token = tokenSeq != null ? tokenSeq.toString() : null;

        if (delimiterCount == 0) {
            base64UrlEncodedHeader = token;
        } else if (delimiterCount == 1) {
            base64UrlEncodedPayload = token;
        }

        delimiterCount++;
        sb.setLength(0);
    } else {
        sb.append(c);
    }
}

```

이부분이 이제 JWT 를 각 . 으로 분리하는 모습입니다 는 마찬가지로 . 인데 이게 . 이 나올때 if 문 안의 로직이 움직이이게 됩니다 그리고 
처음 delimiterCount = 0 인 부분은 header 부분이고 delimiterCount = 1 이면 payload 입니다 그러면 나머지 sign 은 어디에 있을까 ?
사실 이 append 는 먼저 헤더와 , payload 를 분리하고 그떄마다  sb.setLength(0); 를 초기화 하게 됩니다 그럼 제일 마지막에는 sb 에는 sign 이 담기게 되고 


```
if (sb.length() > 0) {
        base64UrlEncodedDigest = sb.toString();
}

```

그 결과는 그 아래 여기에 담기게 됩니다 그러면 일단 각 헤더 payload , sign 은 각 분리가 다 되었습니다 


## 헤더 디코드
```

if (base64UrlEncodedHeader != null) {
    byte[] bytes = base64UrlDecoder.decode(base64UrlEncodedHeader);
    String origValue = new String(bytes, Strings.UTF_8);
    Map<String, Object> m = (Map<String, Object>) readValue(origValue);

    if (base64UrlEncodedDigest != null) {
        header = new DefaultJwsHeader(m);
    } else {
        header = new DefaultHeader(m);
    }

    compressionCodec = compressionCodecResolver.resolveCompressionCodec(header);
}

```

그리고 먼저 헤더를 원래의 값으로 만드는 역활을 합니다 BASE64로 인코딩된것은 다시 디코딩하게 되고 이를 
`Map<String, Object> m = (Map<String, Object>) readValue(origValue);` 를 통해서 Map 타입으로 만들게 됩니다 
그럼 여기 m 에는 

```
m = {LinkedHashMap@7787}  size = 1
 "alg" -> "HS512"

```
이렇게 암호화 방식이 담기게 되고 

## PayLoad 디코드
```

String payload = ""; // https://github.com/jwtk/jjwt/pull/540
if (base64UrlEncodedPayload != null) {
    byte[] bytes = base64UrlDecoder.decode(base64UrlEncodedPayload);
    if (compressionCodec != null) {
        bytes = compressionCodec.decompress(bytes);
    }
    payload = new String(bytes, Strings.UTF_8);
}

```
헤더와 마찬가지로 payload 도 같은 방식으로 decode 하게 됩니다 그럼 이 payload 에는 
`{"sub":"user","jti":"Time","iat":1694315824,"exp":1694315860}` 가 담기게 됩니다 

그리고 이 payload 를 

```

Claims claims = null;

if (!payload.isEmpty() && payload.charAt(0) == '{' && payload.charAt(payload.length() - 1) == '}') { //likely to be json, parse it:
    Map<String, Object> claimsMap = (Map<String, Object>) readValue(payload);
    claims = new DefaultClaims(claimsMap);
}

```

이렇게 Claims 객체에 각 위치에 담기게 됩니다 

## 서명 디코드 

이부분이 제일 핵심입니다 헤더와 , payload 는 충분히 변조가 될 수 있기에 마지막 서명부분으로 이 jwt 가 여기서 발급이 되엇고 유효성 검사를 하게 됩니다 
제일 중요한 부분인 만큼 제일 깁니다 

`if (base64UrlEncodedDigest != null)` 먼저 base64UrlEncodedDigest null 체크를 하게 됩니다 


```

 SignatureAlgorithm algorithm = null;

if (header != null) {
    String alg = jwsHeader.getAlgorithm();
    if (Strings.hasText(alg)) {
        algorithm = SignatureAlgorithm.forName(alg);
    }
}

if (algorithm == null || algorithm == SignatureAlgorithm.NONE) {
    //it is plaintext, but it has a signature.  This is invalid:
    String msg = "JWT string has a digest/signature, but the header does not reference a valid signature " +
        "algorithm.";
    throw new MalformedJwtException(msg);
}

```

그리고 jwsHeader 에서 알고리즘을 뽑아내고 이 알고리즘이 없으면 MalformedJwtException 를 던지게 됩니다 물론 이에 대한 대전제는 base64UrlEncodedDigest != null 이 아닐때 입니다 서멍없이 들어올때는 다른 루트로 jwt 를 뽑아냅니다 

```

if (key != null && keyBytes != null) {
    throw new IllegalStateException("A key object and key bytes cannot both be specified. Choose either.");
} else if ((key != null || keyBytes != null) && signingKeyResolver != null) {
    String object = key != null ? "a key object" : "key bytes";
    throw new IllegalStateException("A signing key resolver and " + object + " cannot both be specified. Choose either.");
}


```

마찬가지로 개인 key 도 존재하는지 살펴보게 됩니다 그게 없으면 역시나 에러를 던지게 됩니다 

```

 if (!Objects.isEmpty(keyBytes)) {

    Assert.isTrue(algorithm.isHmac(),
        "Key bytes can only be specified for HMAC signatures. Please specify a PublicKey or PrivateKey instance.");

    key = new SecretKeySpec(keyBytes, algorithm.getJcaName());
}

```

그리고 이 key 에 개인key(바이트) 그리고 알고리즘 타입을 key 를 넣게 됩니다 

```

String jwtWithoutSignature = base64UrlEncodedHeader + SEPARATOR_CHAR;
if (base64UrlEncodedPayload != null) {
    jwtWithoutSignature += base64UrlEncodedPayload;
}

```

그리고 jwt 는 기존의 헤더와 payload 를 합해서 개인키와 암호화 알고리즘으로 합해젔음으로 다시 jwt 를 조립을 하게 됩니다 그 과정을 보여주는 것이고 

```
JwtSignatureValidator validator;
try {
    algorithm.assertValidVerificationKey(key); //since 0.10.0: https://github.com/jwtk/jjwt/issues/334
    validator = createSignatureValidator(algorithm, key);
}
```
JwtSignatureValidator 안에 알고리즘과 key 를 넣고 createSignatureValidator jwt 를 검증할 수있는 validator 를 만들게 됩니다 하단에서 

```

if (!validator.isValid(jwtWithoutSignature, base64UrlEncodedDigest)) {
    String msg = "JWT signature does not match locally computed signature. JWT validity cannot be " +
        "asserted and should not be trusted.";
    throw new SignatureException(msg);
}

```

하단에서 jwtWithoutSignature 앞에서 다시 조립한 헤더와 , payload 와 base64UrlEncodedDigest 를 비교해서 이 토큰이 올바르게 발급이 되었고 변조가 되었는지 확인을 하게 됩니다 이 부분을 넘어가게 되면 서명부분 jwt 가 변조되지는 않았다는 것입니다 

## 만료시간 체크

```

final Date now = this.clock.now();
long nowTime = now.getTime();

//https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-30#section-4.1.4
//token MUST NOT be accepted on or after any specified exp time:
Date exp = claims.getExpiration();
if (exp != null) {

    long maxTime = nowTime - this.allowedClockSkewMillis;
    Date max = allowSkew ? new Date(maxTime) : now;
    if (max.after(exp)) {
        String expVal = DateFormats.formatIso8601(exp, false);
        String nowVal = DateFormats.formatIso8601(now, false);

        long differenceMillis = maxTime - exp.getTime();

        String msg = "JWT expired at " + expVal + ". Current time: " + nowVal + ", a difference of " +
            differenceMillis + " milliseconds.  Allowed clock skew: " +
            this.allowedClockSkewMillis + " milliseconds.";
        throw new ExpiredJwtException(header, claims, msg);
    }
}

```
이 부분에서는 만료시간을 체크하게 됩니다 이제 클레임에서 lat 와 exp 를 분리해서 시간으 비교하게 됩니다 

이때 if (max.after(exp)) 에서 넘어가게 되면 그 시간의 차이를 분석해서 jwt 만료시간으로 부터 얼만큼 멀어졌는지 보여주게 됩니다 ExpiredJwtException 를 던지게 됩니다 

이제 만료시간까지 넘어가면 이제 완전한 유효성이 검증이 된것이고 

```
Jwt jwt = parse(compact);
```
로 돌아와서 jwt 를 받게 됩니다 

```

header = {DefaultJwsHeader@7752}  size = 1
 "alg" -> "HS512"
body = {DefaultClaims@7754}  size = 4
 "sub" -> "user"
 "jti" -> "Time"
 "iat" -> {Integer@7799} 1694317794
 "exp" -> {Integer@7801} 1694317830
signature = "-TCHwQJDNVaKa-gGz9-AGErJ0yXoK8xnrgeyMVHr2k0IwoXXOQDiaCWE0AySn1cPsuCRcY0xkAxm-frYwzrrQQ"
 value = {byte[86]@7802} [45, 84, 67, 72, 119, 81, 74, 68, 78, 86, 97, 75, 97, 45, 103, 71, 122, 57, 45, 65, 71, 69, 114, 74, 48, 121, 88, 111, 75, 56, 120, 110, 114, 103, 101, 121, 77, 86, 72, 114, 50, 107, 48, 73, 119, 111, 88, 88, 79, 81, 68, 105, 97, 67, 87, 69, 48, 65, 121, 83, 110, 49, 99, 80, 115, 117, 67, 82, 99, 89, 48, 120, 107, 65, 120, 109, 45, 102, 114, 89, 119, 122, 114, 114, 81, 81]
 coder = 0
 hash = 0

```

이런 정보를 담게 되고 이 데이터는 

```

if (jwt instanceof Jws) {
    Jws jws = (Jws) jwt;
    Object body = jws.getBody();
    if (body instanceof Claims) {
        return handler.onClaimsJws((Jws<Claims>) jws);
    } else {
        return handler.onPlaintextJws((Jws<String>) jws);
    }
} else {
    Object body = jwt.getBody();
    if (body instanceof Claims) {
        return handler.onClaimsJwt((Jwt<Header, Claims>) jwt);
    } else {
        return handler.onPlaintextJwt((Jwt<Header, String>) jwt);
    }
}

```

각 클레임으로 분리되어서 반환이 됩니다 

우리는 이렇게 지난시간부터 해서 jwt 발급 및 검증에 대해서 공부를 해보았습니다 