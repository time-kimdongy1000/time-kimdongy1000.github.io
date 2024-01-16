---
title: Spring Secuirty 17 JWT 4 JWT 발급과 검증과정 2
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
클라이언트에서 넘어오는 JWT 는 parserBuilder 를 통해서 유효성을 검토하게 됩니다 

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
DefaultJwtParse 클래스는 jjwt 라이브러리에서 제공하는 JWT 를 파싱할때 사용되는 클래스 입니다


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
## JWS 
JWS 는 Json Web Signature 형식으로 서명된 JWT 를 말합니다 즉 JWS 는 JWT 의 일종으로 헤더와 페이로드로 JWS 를 만들어서 클라이언트에 전달 그리고 전달받은 JWS 를 비교해서 
유효성을 채크합니다 그래서 첫줄에 있는 `jwt instanceof Jws` 이 부분은 현재 들어오는 이 JWT 객체가 Jws 타입인지 체크하고 있는 것입니다 

## parse 
```
@Override
public Jwt parse(String jwt) throws ExpiredJwtException, MalformedJwtException, SignatureException {

    if (this.deserializer == null) {
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

    String payload = ""; 
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

        Key key = this.key;

        if (key == null) { 

            byte[] keyBytes = this.keyBytes;

            if (Objects.isEmpty(keyBytes) && signingKeyResolver != null) { 
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

        String jwtWithoutSignature = base64UrlEncodedHeader + SEPARATOR_CHAR;
        if (base64UrlEncodedPayload != null) {
            jwtWithoutSignature += base64UrlEncodedPayload;
        }

        JwtSignatureValidator validator;
        try {
            algorithm.assertValidVerificationKey(key); 
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

    
    if (claims != null) {

        final Date now = this.clock.now();
        long nowTime = now.getTime();

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
이 부분이 이제 실질적으로 들어오는 JWT 를 분석해서 유효여부를 판단하게 됩니다 



```
Assert.hasText(jwt, "JWT String argument cannot be null or empty.");

if ("..".equals(jwt)) {
    String msg = "JWT string '..' is missing a header.";
    throw new MalformedJwtException(msg);
}

```
먼저 jwt 가 존재하는지 검사하고 jwt 없이 이를 연결하는 .. 만 오게 되는지 찾게 됩니다 그때는 MalformedJwtException 을 던져서 에러를 처리합니다 에러 도 
`String msg = "JWT string '..' is missing a header.";`  이렇게 헤더에서 . 을 찾을 수 없다라고 나옵니다 


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
계속해서 말하지만 JWT 는 .(점)으로 헤더 , 페이로드 , 시그니쳐로 분류가 됩니다 이 문구에서는 처음으로 점이 나온 부분을 헤더 부분 두번째로 점이 나온부분을 페이로드 그리고 그외 나머지를 시그니쳐로 분류를 하고 있습니다  

그러면 헤더는 base64UrlEncodedHeader 담기고 페이로드는 base64UrlEncodedPayload 담기고 나머지 시그니쳐는 sb 에 담기게 됩니다 


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
BASE64로 인코딩된 헤더를 다시 원래대로 디코딩을 하고 있습니다 이렇ㄱ게 되면 map 형태의 key - value 가 반환이 되게 됩니다 이 안에는 현재 알고리즘 타입이 담겨있는 것을 볼 수 있습니다 

```
m = {LinkedHashMap}  size = 1
 "alg" -> "HS512"
```

## 페이로드 디코드
```
String payload = ""; 

if (base64UrlEncodedPayload != null) {
    byte[] bytes = base64UrlDecoder.decode(base64UrlEncodedPayload);
    if (compressionCodec != null) {
        bytes = compressionCodec.decompress(bytes);
    }
    payload = new String(bytes, Strings.UTF_8);
}
```
헤더와 마찬가지로 payload 도 같은 방식으로 decode 하게 됩니다 그럼 이 payload 에는 
```
{
    "sub":"user",
    "jti":"Time",
    "iat":1694315824,
    "exp":1694315860
}
``` 
이렇게 payload 에 담기게 됩니다 


```
Claims claims = null;

if (!payload.isEmpty() && payload.charAt(0) == '{' && payload.charAt(payload.length() - 1) == '}') { //likely to be json, parse it:
    Map<String, Object> claimsMap = (Map<String, Object>) readValue(payload);
    claims = new DefaultClaims(claimsMap);
}
```
이 문구는 payload 가 비어 있지 않고 payload 의 첫번째 글자가 "{" 이며 끝의 글자가 "}" 일때 이 페이로드를 claims 객체에 담아주게 됩니다 이때는 DefaultClaims 로 표준어로 된 
클레임셋으로만 구성이 됩니다 

## 서명 유효성 검사
그리고 JWT 에서 제일 중요한 검사 서명검사입니다 이 부분을 통해서 해당 JWT 의 위변조를 확인해서 위조되었으면 에러를 return 하게 됩니다 

`if (base64UrlEncodedDigest != null)` 먼저 base64UrlEncodedDigest null 체크를 하게 됩니다 이 데이터는 위에서 헤더 페이로드 분리될때 마지막으로 서명부분이 
base64UrlEncodedDigest 변수안으로 들어가게 됩니다 

```
SignatureAlgorithm algorithm = null;

if (header != null) {
    String alg = jwsHeader.getAlgorithm();
    if (Strings.hasText(alg)) {
        algorithm = SignatureAlgorithm.forName(alg);
    }
}

if (algorithm == null || algorithm == SignatureAlgorithm.NONE) {
    String msg = "JWT string has a digest/signature, but the header does not reference a valid signature " +
        "algorithm.";
    throw new MalformedJwtException(msg);
}
```

그리고 jwsHeader 에서 알고리즘을 뽑아내고 이 알고리즘이 없으면 MalformedJwtException 를 던지게 됩니다


```
if (key != null && keyBytes != null) {
    throw new IllegalStateException("A key object and key bytes cannot both be specified. Choose either.");
} else if ((key != null || keyBytes != null) && signingKeyResolver != null) {
    String object = key != null ? "a key object" : "key bytes";
    throw new IllegalStateException("A signing key resolver and " + object + " cannot both be specified. Choose either.");
}
```
마찬가지로 암호화 할떄 사용한 key 가 존재하는지 찾게 됩니다 역시나 key 가 없어도 이는 에러를 뿜게 됩니다 


```
String jwtWithoutSignature = base64UrlEncodedHeader + SEPARATOR_CHAR;
if (base64UrlEncodedPayload != null) {
    jwtWithoutSignature += base64UrlEncodedPayload;
}
```
이는 이제 서명을 서로 비교할려고 BASE64 인코딩된것들을 다시 합치고 있습니다 대상은 헤더와 , 페이로드 입니다 

```
JwtSignatureValidator validator;
try {
    algorithm.assertValidVerificationKey(key); 
    validator = createSignatureValidator(algorithm, key);
}
```

JwtSignatureValidator 안에 알고리즘과 key 를 넣고 createSignatureValidator 함수를 호출해서 유효성검사를 할 수 있는 JwtSignatureValidator 객체를 만들게 됩니다 

```

if (!validator.isValid(jwtWithoutSignature, base64UrlEncodedDigest)) {
    String msg = "JWT signature does not match locally computed signature. JWT validity cannot be " +
        "asserted and should not be trusted.";
    throw new SignatureException(msg);
}

```
그리고 이 위에서 만들어진 유효성 검사기와 , 헤더와 페이로드 조합으로 만든 것과 클라이언트에서 던져진 서명을 isValid 함수를 호출해서 비교를 하게 됩니다 이때 유효하지 않다면 
SignatureException 에러를 뿜고 끝이나게 됩니다 


## 만료시간 체크
```

final Date now = this.clock.now();
long nowTime = now.getTime();

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
JWT 는 단순히 서명의 일치만으로 유효성을 검사하지 않습니다 JWT 안에는 발급시간 표준클레임으로 (iat) 만료시간 표준클레임으로 (exp) 를 포함하고 있습니다 
역시 이 또한 비교해서 만료시간이 현재 시간보다 과거이면 이를 유효하지 않다고 판단해서 ExpiredJwtException 을 발생시키고 끝이나게 됩니다 



그래서 결국 JWT 는 아래와 같은 결과를 얻게 되었습니다 
```
header = {DefaultJwsHeader}  size = 1
 "alg" -> "HS512"
body = {DefaultClaims}  size = 4
 "sub" -> "user"
 "jti" -> "Time"
 "iat" -> {Integer} 1694317794
 "exp" -> {Integer} 1694317830
signature = "-TCHwQJDNVaKa-gGz9-AGErJ0yXoK8xnrgeyMVHr2k0IwoXXOQDiaCWE0AySn1cPsuCRcY0xkAxm-frYwzrrQQ"
```

이번시간에는 JWT 의 검증과정을 하나씩 파해쳐가는 시간을 가져 보았습니다 