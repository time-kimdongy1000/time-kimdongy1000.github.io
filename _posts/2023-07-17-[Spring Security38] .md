---
title: Spring Secuirty 38 React - Spring Security JWT 를 이용한 JWT 토큰 발급2
author: kimdongy1000
date: 2023-07-17 13:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---


어빈시간에는 DB 에 저장된 데이터를 가져오는 것까지 마쳤고 그를 활용해서 Authentication 객체를 만드는것까지 완료했습니다 이번시간에는 이를 활용해서 RSA 기반 JWT 객체를 만들도록 하겠습니다


## LoginService
```

@Autowired
private JWTGenerator jwtGenerator;

@Autowired
private JWK javaWebKey;


public String loginUser(LoginDto loginDto) throws Exception{

    String jwtToken = "";

    Authentication nonAuthentication = new UsernamePasswordAuthenticationToken(loginDto.getEmail() , loginDto.getPassword());
    Authentication authenticatedUser = customAuthenticationService.authenticate(nonAuthentication);

    jwtToken  = jwtGenerator.jwtGenerator(authenticatedUser , javaWebKey);

    return jwtToken;
}


```

지난시간에 Authentication 타입의 authenticatedUser 만들었습니다 이를 활용해서 jwtGenerator.jwtGenerator 활용해서 JWT 토큰을 return 을 하겠습니다

## JWK

```

@Configuration
public class JwtKeyGenerator {

    @Value("${spring.security.keySize}")
    private int keySeize;

    @Value("${spring.security.secretKey}")
    private String secretKey;

    @Bean
    public JWK javaWebKey() throws Exception{

        JWK jwk  = new RSAKeyGenerator(keySeize).keyID(secretKey).algorithm(JWSAlgorithm.RS256).generate();
        return jwk;
    }
}

```

이는 JAVA_WEB_KEY 로 JWT 를 만들때 사용하는 키 입니다 저는 RSA 기반으로 만들것이기 때문에 이렇게 JWK Bean 을 만들어줄것입니다 이를 통해서 JWK Bean을  만들고 이때 
중요한 keySeize , secretKey 를 보게되면

## application.properties
```
spring:
  security:
    secretKey: 4DE6F8A83AB95348D923988F8F82D
    keySize: 2048
    access_tokenExpireTime: 300000
    refresh_tokenExpireTime : 216000000

```

이렇게 등록이 되어있습니다 이를 바탕으로 만들어주고 

## JWTGenerator

```
@Component
public class JWTGenerator {

    @Value("${application.backend.server}")
    private String backend_server;

    @Value("${spring.security.access_tokenExpireTime}")
    private Long access_tokenExpireTime;

    public String jwtGenerator(Authentication authentication , JWK jwtKeyGenerator) throws Exception{

        JWK jwk =  jwtKeyGenerator;

        JWSAlgorithm jwsAlgorithm = (JWSAlgorithm) jwk.getAlgorithm(); //RSA 기반인 알고리즘이 채택 위에서 만든 JWK BEAN JWSAlgorithm.RS256
       
        /*Bean 에서 설정한 KeyId 와 비밀Key 로 각각의 객체를 생성합니다  */
        String keyId = jwk.getKeyID();
        PrivateKey privateKey =  jwk.toRSAKey().toPrivateKey();
        
        /*
        JWT 는 기본적으로 Header , payload , signature(서명)   으로 이루어져 있습니다

        JWSHeader 는 헤더의 정보를 만들어냅니다 헤더는 JWT 를 만든 알고리즘 방법과 KeyId 를 포함합니다

        */
        JWSHeader jwtHeader = new JWSHeader.Builder(jwsAlgorithm).keyID(keyId).build();

        /*

            이 부분이 이제 payload 에 해당하는 부분입니다 JWT 가 표현할려는 정보를 JWTClaimsSet 으로 만들어줍니다
            이때 각각의 정보들은 우리가 만든 인증객체 authentication 에서 가져오고 있습니다
            그리고 JWT 의 만료시간 expirationTime 을 만들고 build 를 하면 payload 완성
        */

        List<String> array_authority =  authentication.getAuthorities().stream().map(x ->{ return x.toString();}).collect(Collectors.toList());


        JWTClaimsSet jwtPayload = new JWTClaimsSet.Builder()
                .subject("user")
                .issuer(backend_server)
                .claim("username" , authentication.getPrincipal())
                .claim("authority" , array_authority)
                //.claim("authority" , "ROLE_MEMBER")
                .expirationTime(new Date(new Date().getTime() +(access_tokenExpireTime)))
                .build();

        /*

            그리고 서명을 만들기 위해서 RSASSASigner  객체를 만들어줍니다
            JWT 서명은 이 JWT 가 위변조 여부를 판단하는 제일 중요한 부분입니다

        */
        RSASSASigner jwsSigner = new RSASSASigner(privateKey);

        /*

            헤더정보와 페이로드 정보를 파탕으로 SignedJWT 객체를 만들어주고 sign 를 호출하면 JWT 토큰이 만들어지게 됩니다

        */

        SignedJWT signedJWT = new SignedJWT(jwtHeader , jwtPayload);

        signedJWT.sign(jwsSigner);

        String token = signedJWT.serialize();

        return token;

    }
}

```
각각의 설명은 지난 RSA 기반으로 만들어진 프로젝트에서도 설명을 한번 했었기 대문에 주석으로 마찬가지로 넘어가도록 하겠습니다 그리고 서명이 마쳐지면 이제 JWT 타입의 String 토큰이 하나 만들어지게 됩니다 

## LoginController
```

@PostMapping("/user/login")
public ResponseEntity<?> loginUser(
    @Valid @RequestBody LoginDto loginDto)
{
    try{

        String jwtToken = loginService.loginUser(loginDto);

        Map<String , Object> result = new HashMap<>();
        result.put("JWT_TOKEN" , jwtToken);

        String to_json_result = gson.toJson(result);


        return new ResponseEntity<>(to_json_result , HttpStatus.OK);
    }catch (Exception e){
        throw new RuntimeException(e);
    }
}


```

그럼이 loginUser 는 이 정보를 바탕으로 JSON_String 타입으로 만들어서 react 로 넘여줍니다

그럼 react 에서는 이 넘어오는 값을

## 결과 저장
```

const api = "/user/login"
const method = "post"
const param = {
    email : email,
    password : password
}

networkApiCall(api , method , param).then(result => {

    if(result.JWT_TOKEN){
        const project_name = f1().project_name;
        localStorage.setItem(`${project_name}_ACCESS_TOKEN` , result.JWT_TOKEN);
        setLogInVaild(true)
    }

}).catch(error => {

    console.log(error)
    setLoginResult(true);
})


```

이렇게 localStorage 에 저장을 하게 됩니다 참고로 이때 만든 networkApiCall 은 제가 공통으로 back-end 와 통신하기 위해서 만든 공통 함수입니다 그냥 참고만 해주세요
이렇게 하면 이제 로그인 정보를 바탕으로 RSA 기반으로 만들어진 토큰 하나가 생성되어서 front-end 로 전달이 됩니다 다음시간에는 이 전달된 토큰을 가지고 Security-context 객체를 만드는 방법을 기술하도록 하겠습니다 




