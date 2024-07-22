---
title: Spring Secuirty 44 Refresh_Token 
author: kimdongy1000
date: 2023-07-17 17:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

## 소스 전체
<https://gitlab.com/kimdongy1000/public_project_amadeus/-/tree/main?ref_type=heads>

해당 소스는 민감한 정보를 제외한 순수 코드입니다 사용하실려면 application.yml 에 자신이 필요한 정보를 기입하시면 사용 가능합니다 
해당 글을 적는부분과 소스의 올라간 부분은 상당히 많이 다릅니다 

그럼 지난시간에 토큰에 관련한 이야기를 조금 했으니 이번시간에는 직접 코드로 한번 만나보겠습니다 stayless 로 개발된 토큰이지만 태생적인 보안의 한계와 서드파티 연계를 위해서 어쩔 수 없이 stayFul 로 돌아가야 합니다 그를 위해서 redis 를 활용할것입니다 redis 는 Key 와 Value 로 저장이 되는 그런 DB 입니다 저는 centos7 환경에서 redis 를 설치를 했습니다 다만 redis 설치에 관련한 정리를 하지 않도록 하겠습니다 



## maven
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

```
먼저 spring boot 에서 redis 를 사용할 수 있게 의존성을 만들어 줍니다 

## application.yml
```

spring:
  datasource:
    redis:
      host : 192.168.40.131
      port : 6379
      password :
 
```

redis 접속정보를 application.properties 정의를 합니다 



## RedisConfig
```
@Configuration
public class RedisConfig {

    @Value("${spring.datasource.redis.host}")
    private String redisHost;

    @Value("${spring.datasource.redis.port}")
    private int redisPort;

    @Value("${spring.datasource.redis.password}")
    private String redisPassword;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {


        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(redisHost);
        redisStandaloneConfiguration.setPort(redisPort);
        redisStandaloneConfiguration.setPassword(redisPassword);

        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration);


        return lettuceConnectionFactory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(){

        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());

        return redisTemplate;

    }

}


```
그리고 그 접속 정보를 바탕으로 redisConfig 를 작성해서 Bean 을 생성합니다 

## RedisRepository

```
@Repository
public class RedisRepository {

    @Autowired
    private RedisTemplate<String , Object> redisTemplate;



    public void insert_redis_data(String key , Object value , long timeOut , TimeUnit timeUnit){
        redisTemplate.opsForValue().set(key , value , timeOut , timeUnit);
    }

    public Object select_redis_data(String key){
        return redisTemplate.opsForValue().get(key);
    }

    public boolean delete_redis_data(String key){
        return redisTemplate.delete(key);
    }

    public boolean expire_redis_data(String key , long timeOut , TimeUnit timeUnit){
        return redisTemplate.expire(key , timeOut , timeUnit);
    }
}


```
그리고 해당 redis Bean 을 가지고 RedisRepository 를 작성해서 간단한 crud 를 할 수 있는 메서드를 만듭니다 


## JWTGenerator
```
@Component
public class JWTGenerator {

    @Value("${application.backend.server}")
    private String backend_server;

    @Value("${spring.security.access_tokenExpireTime}")
    private Long access_tokenExpireTime;

    @Value("${spring.security.refresh_tokenExpireTime}")
    private Long refresh_tokenExpireTime;

    @Autowired
    private RedisRepository redisRepository;

    @Autowired
    private Gson gson;


    public Map<String , Object> jwtGenerator(Authentication authentication , JWK jwtKeyGenerator) throws Exception{

        /*access_token*/
        JWK jwk =  jwtKeyGenerator;

        JWSAlgorithm jwsAlgorithm = (JWSAlgorithm) jwk.getAlgorithm();
        String keyId = jwk.getKeyID();
        PrivateKey privateKey =  jwk.toRSAKey().toPrivateKey();

        JWSHeader access_jwtHeader = new JWSHeader.Builder(jwsAlgorithm).keyID(keyId).build();

        List<String> array_authority =  authentication.getAuthorities().stream().map(x ->{ return x.toString();}).collect(Collectors.toList());


        JWTClaimsSet access_jwtPayload = new JWTClaimsSet.Builder()
                .subject("access_token_user")
                .issuer(backend_server)
                .claim("username" , authentication.getPrincipal())
                .claim("authority" , array_authority)
                //.claim("authority" , "ROLE_MEMBER")
                .expirationTime(new Date(new Date().getTime() +(access_tokenExpireTime)))
                .build();

        RSASSASigner jwsSigner = new RSASSASigner(privateKey);

        SignedJWT access_signedJWT = new SignedJWT(access_jwtHeader , access_jwtPayload);

        access_signedJWT.sign(jwsSigner);

        String access_token = access_signedJWT.serialize();


        /*refresh_token*/
        JWSHeader refresh_JWTHeader = new JWSHeader.Builder(jwsAlgorithm).keyID(keyId).build();
        JWTClaimsSet refresh_jwtPayload = new JWTClaimsSet.Builder().subject("refresh_token_user")
                .issuer(backend_server)
                .claim("username" , authentication.getPrincipal().toString() + "/" + UUID.randomUUID().toString() + "/" + new Date().toString())
                .expirationTime(new Date(new Date().getTime() + (refresh_tokenExpireTime)))
                .build();

        SignedJWT refresh_signedJWT = new SignedJWT(refresh_JWTHeader , refresh_jwtPayload);
        refresh_signedJWT.sign(jwsSigner);

        String refresh_token = refresh_signedJWT.serialize();



        Map<String , Object> result_token = new HashMap<>();
        result_token.put("ACCESS_TOKEN"     , access_token);
        result_token.put("REFRESH_TOKEN"    , refresh_token);


        String refresh_redis_key = refresh_token;
        String refresh_redis_value = gson.toJson(authentication);
        redisRepository.insert_redis_data(refresh_redis_key ,refresh_redis_value , refresh_tokenExpireTime , TimeUnit.MILLISECONDS);

        return result_token;

    }
}

```
최초 일반 로그인 , 또는 구글이나 naver Oauth2 로그인을 할때 이제 access_token 과 더불어서 refresh_token 을 만들어서 return 합니다 이때 

```

String refresh_redis_key = refresh_token;
String refresh_redis_value = gson.toJson(authentication);
redisRepository.insert_redis_data(refresh_redis_key ,refresh_redis_value , refresh_tokenExpireTime , TimeUnit.MILLISECONDS);

```

refresh_token 을 key 로 가지는 authentication 을 저장을 해 둡니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/02834583-4f20-4a12-b044-78934bdb0616) 

그럼 이런 모양으로 redis 에 저장이 됩니다 오른쪽 상단 TTL 이 만료 시간입니다 현재는 약 21만초 (약 3일 정도 되는 시간입니다 )

```


@RestController
public class LoginController {

  
    @GetMapping("/oauth2_{oauth2_service}_login")
    public ResponseEntity<?> google_login(
            @RequestParam("code") String code ,
            @PathVariable("oauth2_service") String oauth2_service,
            HttpServletResponse response

        )
    {
        try{

           Map<String , Object> token = loginService.oauth2_loadByUser(code , oauth2_service);

            String to_json_token_result = gson.toJson(token);
            to_json_token_result = Base64.getUrlEncoder().encodeToString(to_json_token_result.getBytes());


            Cookie cookie = new Cookie("Oauth2_JWT_Token" , to_json_token_result);
            cookie.setMaxAge(60);
            cookie.setPath("/");
            response.addCookie(cookie);

            HttpHeaders headers = new HttpHeaders();
            headers.add(HttpHeaders.SET_COOKIE , cookie.toString());


            return new ResponseEntity<>(null ,headers ,HttpStatus.OK);
        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}


```

이 부분에서 각각의 token 은 Base64 인코딩 과정을 거쳐서 cookie 로 전달이 되게 됩니다 

```

const oauth2_google_login = (event) => {

        const api = "/oauth2/google/oauth2_login_address"
        const method= "get"
        const param = {}

        networkApiCall(api , method , param).then(result => {

            console.log(result)

            
            const options = 'width=700, height=600, top=50, left=50, scrollbars=yes popup=true';
            const google_oauth2_popup = window.open(result.Oauth2_login_url ,'popup' ,  options);
            

           let token_interval =  setInterval(() => {

                const google_jwt_token = getCookie("Oauth2_JWT_Token");

                if(google_jwt_token){
                    const decode_google_jwt_token = commonJsonParser(commonUrlEncoder(google_jwt_token));

                    if(decode_google_jwt_token.ACCESS_TOKEN && decode_google_jwt_token.REFRESH_TOKEN){
                        const project_name = f1().project_name;
    
                        localStorage.setItem(`${project_name}_ACCESS_TOKEN` , `Bearer ${decode_google_jwt_token.ACCESS_TOKEN}`);
                        localStorage.setItem(`${project_name}_REFRESH_TOKEN` , `Bearer ${decode_google_jwt_token.REFRESH_TOKEN}`);
    

                        if(google_oauth2_popup != null ) google_oauth2_popup.close();

                        
                        clearInterval(token_interval)        
                        setLogInVaild(true)
                    }   
                }
            } , 1000)

            
        
        }).catch(error => {
            
            console.log(error)
        })

    }

```

```

localStorage.setItem(`${project_name}_ACCESS_TOKEN` , `Bearer ${decode_google_jwt_token.ACCESS_TOKEN}`);
localStorage.setItem(`${project_name}_REFRESH_TOKEN` , `Bearer ${decode_google_jwt_token.REFRESH_TOKEN}`);

```
받은 토큰은 이와 같이 로컬 스토리지에 저장을 하게 됩니다 


그리고 요청할때는 다음과 같습니다 

```

export async function networkApiCall(api , method , param){

    const headers = new Headers();
    const project_name = f1().project_name;
    
    headers.append("Content-Type" , "application/json");
    

    let access_token = localStorage.getItem(`${project_name}_ACCESS_TOKEN`);
    let refresh_token = localStorage.getItem(`${project_name}_REFRESH_TOKEN`);

    if(access_token) access_token =  access_token.substring(7);
    if(refresh_token) refresh_token = refresh_token.substring(7);

    

    if(access_token) headers.append("Authorization" , `Bearer ${access_token}`);
    if(refresh_token) headers.append("Authorization_Refresh" , `Bearer ${refresh_token}`)
    

    let options = {

        headers : headers,
        url : f1().api_base_url + api ,
        method : method
    }

    if(!("get"== method.toLowerCase("get")) && param) options.body = JSON.stringify(param);

    return await fetch(options.url , options).then( (response) => {

        

        

        if(response.status === 401){

            const project_name = f1().project_name;
            localStorage.removeItem(`${project_name}_ACCESS_TOKEN`)
            localStorage.removeItem(`${project_name}_REFRESH_TOKEN`)
            window.location.href = '/login';
            
            
        }else {

            const access_token = response.headers.get('Authorization');
            const refresh_token = response.headers.get('Authorization_Refresh');

            if(access_token) localStorage.setItem(`${project_name}_ACCESS_TOKEN` , access_token);
            if(refresh_token) localStorage.setItem(`${project_name}_REFRESH_TOKEN` , refresh_token);


            return response.json();
        }
        
        
    
    }).then( (response2) => {


        if(response2.HttpStatus === 500){        
            throw new Error(response2.errorMsg);
        }

        
                
        return response2
    })
}

```
다시 요청을 할때는 이와 같이 다시 앞에 있는 이 토큰을 헤더에 심어서 보내게 됩니다 


## CustomJwtAuthenticationFilter
```
@Slf4j
public class CustomJwtAuthenticationFilter extends OncePerRequestFilter {

    private CustomJWTParser jwtParser;

    private JWK jwk;

    private JWTGenerator jwtGenerator;

    private Gson gson;

    private Long access_tokenExpireTime;

    private RedisRepository redisRepository;

    private String return_product_mode;



    public CustomJwtAuthenticationFilter(CustomJWTParser jwtParser  , JWK jwk , JWTGenerator jwtGenerator , Gson gson  , RedisRepository redisRepository , Long access_tokenExpireTime , String return_product_mode) {
        this.jwtParser = jwtParser;
        this.jwk = jwk;
        this.jwtGenerator = jwtGenerator;
        this.gson = gson;
        this.redisRepository = redisRepository;
        this.access_tokenExpireTime = access_tokenExpireTime;
        this.return_product_mode = return_product_mode;
    }








    private static final String[] ALLOW_URL = {
                "/user/register"
                , "/user/login"
                , "/oauth2/google/oauth2_login_address"
                , "/oauth2_google_login"
                , "/oauth2/naver/oauth2_login_address"
                , "/oauth2_naver_login"


                ,"/favicon.ico"
    };

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        String authorizationHeader = request.getHeader("Authorization");
        String authorizationHeader_refresh = request.getHeader("Authorization_Refresh");
        final String url = request.getRequestURI();
        String method = request.getMethod();

        List<String> allow_url_matcher = Arrays.stream(ALLOW_URL).filter(x -> url.equals(x)).collect(Collectors.toList());


        if (allow_url_matcher.size() > 0 || "OPTIONS".equals(method.toUpperCase())) {
            filterChain.doFilter(request, response);
        } else {

            if (authorizationHeader != null && authorizationHeader.startsWith("Bearer ") && authorizationHeader_refresh != null && authorizationHeader_refresh.startsWith("Bearer ")) {

                String accessToken = authorizationHeader.substring(7); // "Bearer " 다음의 값만 추출 access_token
                String refreshToken = authorizationHeader_refresh.substring(7); // "Bearer " 다음의 값만 추출 refresh_token


                try {

                    RSASSAVerifier rsassaVerifier = new RSASSAVerifier((RSAKey) jwk.toPublicJWK());
                    SignedJWT access_token_signedJWT = SignedJWT.parse(accessToken);
                    boolean access_verify = access_token_signedJWT.verify(rsassaVerifier);

                    Authentication authentication = null;


                    if(!access_verify){
                        throw new SignatureException("잘못된 서명입니다.");
                    }

                    JWTClaimsSet access_token_jwtClaimsSet = access_token_signedJWT.getJWTClaimsSet();

                    Date jwt_access_token_expire_time =  access_token_jwtClaimsSet.getExpirationTime();
                    Date today = new Date();
                    Date today_add_expire_time = new Date(today.getTime());

                    /*access_token 이 만료 시간이 되었다면 이제는 refreshP_toekn 으로 위임*/

                    if(jwt_access_token_expire_time.getTime() <= today_add_expire_time.getTime()){

                        /*
                        
                          refresh_token 에 관련한 작업을 진행을 하게 됩니다 다만 과정은 위에 있는 access_token 과 일치합니다 
                          서명 체크와 , 시간체크 이때도 만약 이 검증을 통과하지 못한다면 이제는 401 에러로 return 하고 사용자는 다시금 로그인 과정을 거쳐야 합니다 

                        */
                        SignedJWT refresh_token_signedJWT = SignedJWT.parse(refreshToken);
                        boolean refresh_token_verify = refresh_token_signedJWT.verify(rsassaVerifier);

                        if(!refresh_token_verify){
                            throw new SignatureException("잘못된 서명입니다.");
                        }

                        JWTClaimsSet refresh_token_jwtClaimsSet = refresh_token_signedJWT.getJWTClaimsSet();
                        Date jwt_refresh_token_expire_time =  refresh_token_jwtClaimsSet.getExpirationTime();

                        if(jwt_refresh_token_expire_time.getTime() <= today_add_expire_time.getTime()){
                            throw new RuntimeException("시간이 만료된 토큰입니다.");
                        }

                        /*

                          그리고 refresh_token 으로 저장한 UserDetails 정보를 가져오게 됩니다 
                          그리고 mode 에 따라서 키를 삭제 할지 아니면 30초 후에 해당 key 를 만료시키는 방법으로 코딩을 했습니다 해보니 후자가 훨씬 안정적으로 key 를 관리 할 수 있습니다 

                        */
                        String saved_refresh_token_authentication = redisRepository.select_redis_data(refreshToken).toString();

                        if("Y".equals(return_product_mode)){
                            boolean is_del_token =  redisRepository.delete_redis_data(refreshToken);
                            log.info("{}" , is_del_token);

                        }else{
                            boolean is_expired_token = redisRepository.expire_redis_data(refreshToken , 30  , TimeUnit.SECONDS);
                            log.info("{}" , is_expired_token);

                        }


                        /*
                        
                          그럼 이곳에서는 다시 인증을 한 userDetails 정보가 아닌 redis 에 저장된 정볼르 바탕으로 Authentication 토큰을 만들게 됩니다 

                        */
                        AuthenticationDto refresh_authentication = gson.fromJson(saved_refresh_token_authentication , AuthenticationDto.class);

                        List<GrantedAuthority> array_authority = refresh_authentication.getAuthorities().stream().map(x -> new SimpleGrantedAuthority(x.get("role").toString())).collect(Collectors.toList());
                        authentication = new UsernamePasswordAuthenticationToken(refresh_authentication.getPrincipal() , refresh_authentication.getCredentials() , array_authority);


                        /*
                        
                          해당 authentication 바탕으로 다시 토큰을 만들고 (access_token , refresh_token 이때 요청으로 넘어온 refresh_token 은 삭제 또는 만료)
                          여기서는 새롭게 발급받은 refresh_token 을 다시 저장하고 해당 토큰을 response 객체에 담아서 return 을 해주게 됩니다 

                        */
                        Map<String , Object> result_token = jwtGenerator.jwtGenerator(authentication , jwk);

                        response.setHeader("Authorization" , "Bearer " + result_token.get("ACCESS_TOKEN"));
                        response.setHeader("Authorization_Refresh" , "Bearer " + result_token.get("REFRESH_TOKEN"));


                        /*

                          refresh_token 으로 만들어진 정보를 가지고 setAuthentication 정보를 가지고 인증을 지속적으로 유지를 시킵니다 

                        */

                        SecurityContextHolder.getContext().setAuthentication(authentication);
                        filterChain.doFilter(request, response);

                        return;
                    }

                    String username = (String)access_token_jwtClaimsSet.getClaim("username");
                    List<String> authority = (List<String>) access_token_jwtClaimsSet.getClaim("authority");


                    List<GrantedAuthority> array_authority = authority.stream().map(x -> new SimpleGrantedAuthority(x)).collect(Collectors.toList());

                    UserDetails userDetails = new User(username , UUID.randomUUID().toString() , array_authority);
                    authentication = new UsernamePasswordAuthenticationToken(userDetails , null , array_authority);
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                    filterChain.doFilter(request, response);



                } catch (Exception e) {
                    logger.error("===================================");

                    logger.error(e);

                    logger.error("===================================");
                    response.setStatus(HttpStatus.UNAUTHORIZED.value());
                    response.getWriter().write("UnAuthorized User");
                }
            } else {
                response.setStatus(HttpStatus.UNAUTHORIZED.value());
                response.getWriter().write("UnAuthorized User");


            }
        }
    }
}


```
이렇게 해서 우리는 access_token 과 그것을 뒷받침하는 refresh_token 을 만들어보고 우리만의 정책을 만들어서 코드 배포를 해보았습니다 원래는 이 소스 코드들은 비공개 예정이었지만 
중간중간 변화된 값이 많기 때문에 중요한 정보는 지워놓고 back-end 소스코들르 배포 하도록 하겠습니다 (조만간 주소를 만들어서 해당 글 및 해당 모든 글 최상단에 적도록 하겠습니다)
이것으로 우리는 우리만의 refresh_token 을 만들어보고 전략을 세우는 시간을 가져보았습니다 
