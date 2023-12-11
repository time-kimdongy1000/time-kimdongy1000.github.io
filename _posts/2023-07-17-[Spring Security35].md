---
title: Spring Secuirty 35 RSA 기반 Authentication 프로젝트
author: kimdongy1000
date: 2023-07-17 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

앞전에는 MAC 기반의 대칭키의 형태로 JWT 를 만들었다면 이번시간에는 RSA 비대칭키 기반으로 한번 암호화 복호화를 해보겠습니다 회원가입부터 JWT 복호화 까지는 전부 일치하는데 
부분부분 MAC 을 사용한 부분만 수정하도록 하겠습니다 

## git 소스
https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_mac_authentication_project?ref_type=heads

## JwtKeyGenerator

```
@Configuration
public class JwtKeyGenerator {

    /*
    * JwtKeyGenerator 역활은 여기서는 RS256 방식의 비대칭키를 만드는 곳입니다 다른곳에서 이 RSAKey 를 Bean 으로 받아서 사용할 예정
    *
    * 기본적으로 RSAKey 는 RSAKeyGenerator 로 생성하게 되는데
    * keySeize 같은경우는 제일 작은 값이 2048 크면 클수록 보안에 유리
    * KeyID 같은 경우는 key 를 식별하는 고유 값
    * 그리고 알고리즘은 RS 관련한 알고리즘으로 만들어줍니다
    *
    *
    * */

    @Value("${spring.security.keySize:2048}")
    private int keySeize;

    @Value("${spring.security.secretKey:application}")
    private String secretKey;


    @Bean
    public RSAKey createToken() throws Exception{
        RSAKey rsaKey = new RSAKeyGenerator(keySeize).keyID(secretKey).algorithm(JWSAlgorithm.RS256).generate();
        return rsaKey;

    }

}


```
기본적으로 암호화 복호화 할때 사용하는 key 가 바뀌게 됩니다 앞에서는 앞에서는 대칭키를 만드는 OctetSequenceKey 였다면 비대칭키는 RSAKey 로 만들어지게 됩니다 
이떄 keySeize 같은 경우는 2048이 제일 최소값입니다 (MAC 기반일떄는 256) 이 값은 크면 클수록 보안에 유리해집니다 

## JwtGenerator
```


public class JwtGenerator {

    public String jwtGenerator(UserDetails userDetails , RSAKey jwtKeyGenerator) throws Exception{

        /*
         * RSA 는 JWK 세트에 속하는 키 중 하나로 비대칭키를 표현할때 사용하게 됩니다
         * JWK Json Web Key 디지털 서명을 위한 키를 나타내는 표준입니다
         * 암호화와 복호화가 서로다른 key 이며 이때 공개key 로 암호화를 하고 privte key 로 복호화를 하게 됩니다 이는 역방향으로 가능합니다
         * 지금 예제는 private key 로 암호화를 하고 publickey 로 복호화를 할 예정입니다 이 방식은 주로 디지털 서명을 할떄 사용하고
         * 은행에서 여러분의 비밀번호를 암호화 할때는 반대로 하게 됩니다 (공개키로 암호화를 하고 비공개키로 복호화를 하게 됩니다 )
         */
        RSAKey rsaKey =  jwtKeyGenerator;


        /*
        * RsaKey 는 기본적으로 만들때
        * keySize 와 private key 및 알고리즘을 지정해서 비대칭키를 만들 수 있습니다
        * KeySize 는 key 길이를 나타내는것으로 비대칭키는 보안을 위해서 2048bit를 권장하고 있습니다
        * algorithm 는 암호화 할때 사용하는 어떤 방식의 알고리즘을 사용할지 정하게 됩니다
        *
        * PrivateKey RSA 에서 비공개키를 가져오는 모습입니다
        *
        * */
        JWSAlgorithm jwsAlgorithm = (JWSAlgorithm) rsaKey.getAlgorithm();
        String keyId = rsaKey.getKeyID();
        PrivateKey privateKey =  rsaKey.toPrivateKey();


        List<String>  authorities = userDetails.getAuthorities().stream().map( x-> {return x.getAuthority();}).collect(Collectors.toList());
        authorities.add("EMAIL");
        authorities.add("PROFILE");


        /* JWSHeader 은 JWT 를 만들떄 헤더에 속하는 데이터로 이때는 알고리즘과 를 포함한 값의 길이를 반환합니다
         *
         */
        JWSHeader jwtHeader = new JWSHeader.Builder(jwsAlgorithm).keyID(keyId).build();

        /*
        * JWTClaimsSet 는 payload 를 나타내는것으로 이에 대해서는 앞전에 한번 설명드린적이 있으므로 pass 하겠습니다
        *
        * */

        JWTClaimsSet jwtPayload = new JWTClaimsSet.Builder()
                .subject("user")
                .issuer("httpL//localhost:8080")
                .claim("username" , userDetails.getUsername())
                .claim("authority" , authorities)
                .expirationTime(new Date(new Date().getTime() + 60 * 1000 * 5))
                .build();

        /*
        * 그리고 이 부분이 서명부분이다 RSASSASigner 부분으로 privateKey 서명을 만들고
        * */
        RSASSASigner jwsSigner = new RSASSASigner(privateKey);

        /*
        * JWT 를 서명할때는 헤더와 payload 가 필요함으로 SignedJWT 객체에 첨부후
        *
        * */
        SignedJWT signedJWT = new SignedJWT(jwtHeader , jwtPayload);

        /*
        * 만든 비대칭 key 그중에서 private key 로 서명을 하게 됩니다  로 서명을 하게 됩니다
        *
        * */
        signedJWT.sign(jwsSigner);

        String token = signedJWT.serialize();

        return token;
    }
}


```
로그인을 이 완료되었을때 호출되는 onAuthenticationSuccess 안에서 호출되는 부분으로 JWT 를 만들때 RSA 를 이용해서 인코딩 하는 모습을 보고 있습니다 

```
JWSAlgorithm jwsAlgorithm = (JWSAlgorithm) rsaKey.getAlgorithm();
String keyId = rsaKey.getKeyID();
PrivateKey privateKey =  rsaKey.toPrivateKey();
```
이 부분을 보게 주석에도 달아놓았지만 디지털서명 (JWT) 같은 것은 지금처럼 private - key 로 암호화를 해서 나가게 됩니다 반대로 은행같은 경우는 공개키로 암호화를 한뒤 
서버에서 private - key 로 복호화 하게 됩니다 그래서 언제 어디서 쓰냐에 따라서 암호화를 private key 로 할것인지 public 로 할것인지 결정하게 됩니다 

서명하는 부분을 보게 되면 

```

RSASSASigner jwsSigner = new RSASSASigner(privateKey);
SignedJWT signedJWT = new SignedJWT(jwtHeader , jwtPayload);
signedJWT.sign(jwsSigner);


```
이 모습을 보게 되는데 RSASSASigner 비대칭키를 서명하기 위한 객체로 이때는 privateKey 를 사용하게 됩니다 그리고 MAC 와 마찬가지로 SignedJWT 객체를 만들어서 객체를 생성하고 
이를 RSASSASigner 를 통해서 서명을 하게 됩니다 


## JwtAuthenticationFilter

```
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private RSAKey jwtKeyGenerator;

    public JwtAuthenticationFilter(RSAKey jwtKeyGenerator){
        this.jwtKeyGenerator = jwtKeyGenerator;
    }



    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {


        /*
        * 쿠키 탐색 쿠키가 없으면 그냥 다음 filter 통과
        * */
        Cookie[]  cookies = request.getCookies();
        if(cookies == null ){
            filterChain.doFilter(request , response);
            return;
        }

        /*
        * 쿠키 탐색을 하면서 우리가 앞에서 넣어주는 쿠를 찾게 됩니다
        *
        * */
        for (Cookie cookie : cookies) {
            String request_cookie_name = cookie.getName();

            if("jwtToken".equals(request_cookie_name)){

                /*
                * 쿠키에서 값을 분리
                *
                * */
                String token =  cookie.getValue();

                try{

                    /* bean 으로 등록한 RSA 기반의 JWK 를 불러와서 넣음
                     *
                     * */
                    RSAKey rsaKey = jwtKeyGenerator;

                    /*
                       token 을 parse 로 넣어서 서명을 검토하기 위한 SignedJWT 객체를 만들게 됩니다

                     */
                    SignedJWT signedJWT = SignedJWT.parse(token);

                    /*
                    * Bean 으로 등록한 JWK 를 기반으로 서명이 일치하는지 아닌지 확인을 하기 위해서
                    * RSASSAVerifier 객체를 만들게 됩니다 이때 MACSinger 는 secret 를 넣어서 생성하는 반면
                    * 지금과 같은 디지털 서명부분에는 공개를 key 를 넣어서 일치하는지 확인을 하게 됩니다
                    * */
                    RSASSAVerifier rsassaVerifier = new RSASSAVerifier(rsaKey.toRSAPublicKey());

                    /*
                    * 그리고 JWT 의 서명부분과 private key 를 넣은 RSASSAVerifier 를 통해서 verify 를 불러오게 넣게 되면
                    * true , false 를 반환하게 됩니다
                    *
                    * */
                    boolean verify = signedJWT.verify(rsassaVerifier);

                    /*
                    * true 가 되면 여기서 발급한 jwt 가 맞기 때문에
                    * 이제 인증과정으로 가게 됩니다
                    *
                    * */
                    if(verify){

                        /*
                        *
                        * JWTClaimsSet 으로 값을 분리
                        *
                        * */
                        JWTClaimsSet jwtClaimsSet = signedJWT.getJWTClaimsSet();
                        String username = (String)jwtClaimsSet.getClaim("username");
                        List<String> authority = (List<String>) jwtClaimsSet.getClaim("authority");

                        if(username == null) throw new UsernameNotFoundException("username 을 찾을 수 없습니다");
                        if(authority.isEmpty()) throw new RuntimeException("등록된 권한이 없습니다");

                        List<GrantedAuthority> array_authority = authority.stream().map(x -> new SimpleGrantedAuthority(x)).collect(Collectors.toList());

                        /*
                        * JWT 에서 분리한 값에 username , authority 로 새로운 UserDetails 를 만들게 됩니다
                        * 이때 비밀번호는 없기 때문에 임시 비밀번호를 발급해서 새로운 User 를 넣게 되고
                        *
                        * */
                        UserDetails userDetails = new User(username , UUID.randomUUID().toString() , array_authority);

                        /*
                        * 인증객체를 UsernamePasswordAuthenticationToken 넘겨서 인증을 받게끔 위임합니다
                        *
                        * */

                        Authentication authenticationToken = new UsernamePasswordAuthenticationToken(userDetails , null , array_authority);

                        /*
                        * 그리고 인증이 완료되면 Authentication 객체를 SecurityContextHolder 심는것으로 끝입니다
                        * */

                        SecurityContextHolder.getContext().setAuthentication(authenticationToken);

                        filterChain.doFilter(request , response);
                        return;
                    }
                }catch(Exception e){
                    throw new RuntimeException(e);
                }
            }
        }

        filterChain.doFilter(request , response);
        return;
    }
}

```

쿠키에서 JWT 값을 가져오는것은 동일한 로직인데 이떄 다음을 보게 되면 

```

RSASSAVerifier rsassaVerifier = new RSASSAVerifier(rsaKey.toRSAPublicKey());
boolean verify = signedJWT.verify(rsassaVerifier);

```
비대칭키를 복호화할때는 RSASSAVerifier 를 사용해서 복호활르 진행을 하게 됩니다 이때 rsaKey 의 공개를 key 를 이용해서 서명이 일치하는지 확을 하게 되고 
마찬가지로 verify true 가 나오게 되면 인증 완료 다음로직은 똑같습니다 

이렇게 해서 우리는 MAC 기반의 대칭key 로 JWT 를 서명하고 Verifier 를 하는 것을 보았고 비대칭key 기반의 RSA 로 서명을 하고 Verifier 하는거 까지 보았다 
정리를 해보면

MAC 기반은 대칭key 기반으로 JWT 를 만드는데 이떄 secret - key 가 필요하고 알고리즘은 HS기반의 알고리즘에 서명을 만들때에는 MACSigner 를 사용해서 서명을 하고 
복호화를 할떄 서명이 일치하는지는 MACVerifier 를 사용해서 서명이 일치하는지 보게 됩니다 

RSA 기반은 비대칭key 를 기반으로 JWT 를 만드는데 이때 지금과 같은 디지털 서명부분은 private - key 로 암호화를 하고 알고리즘은 RS 기반의 알고리즘으로 만들게 됩니다 
그리고 서명을 할때에는 RSASSASigner 를 사용해서 서명을 하고 서명이 일치할때 보는것은 RSASSAVerifier 를 통해서 서명이 일치하는지 확인을 하게 됩니다 

