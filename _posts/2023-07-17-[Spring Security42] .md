---
title: Spring Secuirty 42 RestApi 기반 naver OAuth2 인증 만들기 
author: kimdongy1000
date: 2023-07-17 16:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

## 소스 전체
<https://gitlab.com/kimdongy1000/public_project_amadeus/-/tree/main?ref_type=heads>

해당 소스는 민감한 정보를 제외한 순수 코드입니다 사용하실려면 application.yml 에 자신이 필요한 정보를 기입하시면 사용 가능합니다 
해당 글을 적는부분과 소스의 올라간 부분은 상당히 많이 다릅니다 

지난시간에는 google 을 이용한 oauth2 로그인 주소를 만드는것부터 해서 인가코드를 받고 인가코드를 서버에서 처리해서 access_token 을 받아서 그 내용을 가지고 OIDC 인증 완료 아니면 다시 access_token 으로 userInfo url 을 호출해서 구글에 해당 사용자의 user 정보를 가져와서 UserDetails 을 만들어서 우리 방식의 JWT 토큰을 만든다음 클라이언트로 보내는 방식까지 구현을 해보았습니다 이번시간에는 naver Ouaht2 방식으로 마찬가지로 적어나가겠습니다

## naver 로그인 api 주소
```

https://developers.naver.com/docs/login/api/api.md

```
naver api 를 받기 위한 주소입니다 설정에 대해서는 적지 않겠습니다 검색을 해보시면 많은 분들꼐서 적어 두셨고 저는 코드에만 집중을 하겠습니다 


## application.yml

```
spring:
    security:
        naver_client_id :
        naver_client_secret :
        naver_redirect_url : http://localhost:8080/oauth2_naver_login
        naver_authorization_url : https://nid.naver.com/oauth2.0/authorize
        naver_token_url : https://nid.naver.com/oauth2.0/token
        naver_userInfo_url : https://openapi.naver.com/v1/nid/me

```

먼저 naver 에서 발급받은 client_id , client_secret 를 받습니다 참고로 google 같은 대표적인 인가서버는 CommonOAuth2Provider 에 인증/인가에 필요한 내용이 들어가 있지만 naver 같은 경우는 그렇지 않기 때문에 우리가 입력을 해주어야 합니다 그래서 최소한의 정보 naver_authorization_url , naver_token_url , naver_userInfo_url 을 작성을 해줍니다

## Oauth2Config

```
@Configuration
public class Oauth2Config {

@Value("${spring.security.naver_client_id}")
    private String naver_client_id;

    @Value("${spring.security.naver_client_secret}")
    private String naver_client_secret;

    @Value("${spring.security.naver_redirect_url}")
    private String naver_redirect_url;

    @Value("${spring.security.naver_authorization_url}")
    private String naver_authorization_url;

    @Value("${spring.security.naver_token_url}")
    private String naver_token_url;

    @Value("${spring.security.naver_userInfo_url}")
    private String naver_userInfo_url;



    private ClientRegistration naverClientRegistration(){

        return ClientRegistration.withRegistrationId("naver")
                .clientId(naver_client_id)
                .clientSecret(naver_client_secret)
                .redirectUri(naver_redirect_url)
                .authorizationUri(naver_authorization_url)
                .tokenUri(naver_token_url)
                .userInfoUri(naver_userInfo_url)
                .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
                .build();

    }

    @Bean
    public ClientRegistrationRepository clientRegistrationRepository(){
        ClientRegistrationRepository clientRegistrationRepository = new InMemoryClientRegistrationRepository(googleClientRegistration() , naverClientRegistration());
        return clientRegistrationRepository;

    }

}


```

여기서는 앞에서 보았던것처럼 우리가 어디에서든 인가접속정보를 활용하기 위해 해당 정보를 ClientRegistration 에 저장하는 것입니다 naver 저장을 위해서 `naverClientRegistration()` 을 만들어 놓고 인메모리 데이터베이스에 저장을 하겠습니다

## naver 로그인 주소생성

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7dd0e79d-598c-46ba-adc9-96ad639964c3)

이 상황입니다 메인에서 네이버 로그인 버튼을 클릭했을 서버로 가서 주소를 만들어서 클라이언트로 던지고 클라이언트는 이 내용을 바탕으로 팝업창을 만드는것입니다


```

 const oauth2_naver_login = (event) => {

        const api = "/oauth2/naver/oauth2_login_address"
        const method= "get"
        const param = {}

        networkApiCall(api , method , param).then(result => {

            console.log(result);

            
             const options = 'width=700, height=600, top=50, left=50, scrollbars=yes popup=true';
             const naver_oauth2_popup = window.open(result.Oauth2_login_url ,'popup' ,  options);

             
            

             let token_interval = setInterval(() => {

                 const naver_jwt_token = getCookie("Oauth2_JWT_Token");

                 

                 if(naver_jwt_token){
                    const decode_naver_jwt_token = commonJsonParser(commonUrlEncoder(naver_jwt_token));
                    if(decode_naver_jwt_token.ACCESS_TOKEN && decode_naver_jwt_token.REFRESH_TOKEN){
                        const project_name = f1().project_name;
                        localStorage.setItem(`${project_name}_ACCESS_TOKEN` , decode_naver_jwt_token.ACCESS_TOKEN);
                        localStorage.setItem(`${project_name}_REFRESH_TOKEN` , decode_naver_jwt_token.REFRESH_TOKEN);
                        naver_oauth2_popup.close();
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
이 부분은 이전 google 로그인 팝업을 만드는 부분하고 동일 합니다 버튼을 클릭하면 `const api = "/oauth2/naver/oauth2_login_address"` 요청으로 서버로 날려서 주소를 가져옵니다 그럼 서버에서 주소를 만들어준다음 result 반환을 하고 이 내용으로 팝업을 만들어서 렌더링 하는 것입니다

## LoginController

@RestController
public class LoginController {

    @Autowired
    private Gson gson;

    @Autowired
    private ClientRegistrationRepository clientRegistrationRepository;  

    @GetMapping("/oauth2/{oauth2_service}/oauth2_login_address")
    public ResponseEntity<?> google_oauth2_address(
            @PathVariable("oauth2_service") String oauth2_service
    )

    {
        try{

            ClientRegistration clientRegistration = null;
            StringBuffer return_url = new StringBuffer();

            switch (oauth2_service) {

                case "naver" :
                    clientRegistration = clientRegistrationRepository.findByRegistrationId("naver");
                    return_url.append(clientRegistration.getProviderDetails().getAuthorizationUri());
                    return_url.append("?");
                    return_url.append("response_type=code");
                    return_url.append("&");
                    return_url.append("client_id=");
                    return_url.append(clientRegistration.getClientId());
                    return_url.append("&");
                    return_url.append("state=");
                    return_url.append("STATE_URL");
                    return_url.append("&");
                    return_url.append("redirect_uri=");
                    return_url.append(clientRegistration.getRedirectUri());


                    break;
            }

            Map<String , Object> result = new HashMap<>();
            result.put("Oauth2_login_url" , return_url);

            return new ResponseEntity<>(gson.toJson(result) , HttpStatus.OK);

        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }

}

그러면 요청은 서버의 이 핸들러로 들어오게 되는데 이때 우리가 clientRegistrationRepository 에 저장해둔 저장소에 가서 naver key 가 naver 인 ClientRegistrarion 을 가져오고 위와 같은 코드로 인가코드를 받을 수 있는 주소를 클라이언트로 return 을 해줍니다
참고로 naver oauth2 로그인은 state 가 필수값입니다 원래는 랜덤한 내용의 문자열을 지속해서 주고받아야 하는데 일단은 하드코딩 후 진행을 하겠습니다

그리고 로그인을 진행을 하게 되면

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
이쪽으로 들어오게 됩니다 이는 google_redirect 와 동일한 주소입니다 이때 동적으로 주소를 받기 위해서 `{oauth2_service}` 핸들러 이 부분을 PathVariable 로 작성을 했습니다 그럼 동일하게 oauth2_loadByUser 로 넘어가게 됩니다 이때 code 와 oauth2_service 파라미터를 같이 넘기게 됩니다


## LoginService
```
@Service
public class LoginService {

    @Autowired
    private LoginRepository loginRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private CustomAuthenticationService customAuthenticationService;

    @Autowired
    private JWTGenerator jwtGenerator;

    @Autowired
    private JWK javaWebKey;

    @Autowired
    private Gson gson;


    @Autowired
    private ClientRegistrationRepository clientRegistrationRepository;


   
    public Map<String, Object> oauth2_loadByUser(String code , String oauth_service) throws Exception{


        ClientRegistration clientRegistration = null;


        String client_id = null;
        String client_secret = null;
        String redirect_url = null;
        String token_url = null;
        String user_info = null;

        RestTemplate restTemplate = new RestTemplate();
        Map<String , Object> params = new HashMap<>();

/*

case - when 으로 작성해서 하나의 메서드에 여러 oauth2 인증/인가를 넣었습니다

*/

        switch (oauth_service) {
           

            case "naver":

/*

Bean 으로 저장한 clientRegistrationRepository key 값을 naver 로 찾아서 clientRegistration 객체를 만들어줍니다 그 안에서

        client_id
        client_secret
        redirect_url
        token_url
        user_info

을 참조해서 가져옵니다 
*/
                clientRegistration = clientRegistrationRepository.findByRegistrationId("naver");

                client_id = clientRegistration.getClientId();
                client_secret = clientRegistration.getClientSecret();
                redirect_url = clientRegistration.getRedirectUri();
                token_url = clientRegistration.getProviderDetails().getTokenUri();
                user_info = clientRegistration.getProviderDetails().getUserInfoEndpoint().getUri().toString();

/*

그리고 google 인증때에는 responseEntity 로 post 전송을 했는데
naver 전송은 queryParam 으로 해당 데이터를 세팅후에 token_url 을 통해서 token 을 호출합니다

*/

                UriComponentsBuilder naver_access_token_builder = UriComponentsBuilder.fromHttpUrl(token_url)
                        .queryParam("client_id" , client_id)
                        .queryParam("client_secret" , client_secret)
                        .queryParam("code" , code)
                        .queryParam("state" , "STATE_URL")
                        .queryParam("grant_type" , "authorization_code");

                String naver_token_url  = naver_access_token_builder.toUriString();




/*

그리고 get 요청으로 토큰을 가져옵니다

*/
                ResponseEntity<String> naver_access_token_responseEntity = restTemplate.getForEntity(naver_token_url , String.class);
                if(naver_access_token_responseEntity.getStatusCode() != HttpStatus.OK){
                    throw new RuntimeException("네이버 로그인에 실패했습니다.");
                }

/*

토큰의 정보를 제가 만든 POJO 클래스로 변환을 해줍니다 그럼 이 POJO 클래스에는

result = {NaverOauthTokenDto@13888}
access_token = "AAAAPQ1rXLZGy5Q3dz2Un8V1JgmaVmshm3kYxJ8wBw19Ag_-KXwjQuEFKvdutpP1dmr4CLb83lzJmSeUwV1w8suuIac"
refresh_token = "vNwZMOIHPRqZf1ipyqsXgDVon2visVbVmjCU4coDisoC3aNisvxzC8Misj3hTp0QiiWtUSGA2iihRipCeCCj29ipplIzrf4Ttj19Ca64LiinYHipQWWyipBV2xcpkbFS9G49UnyOrC5y"
token_type = "bearer"
expires_in = 3600

이와 같은 정보가 담기게 됩니다

*/

                NaverOauthTokenDto naverOauthTokenDto = gson.fromJson(naver_access_token_responseEntity.getBody() , NaverOauthTokenDto.class);

/*

naver 는 oidc 인증이 없기 때문에 다시 access_token 처럼 다시 한번 서버로 요청을 해서 사용자의 정보를 가져와야 합니다
방법은 google userInfo 요청처럼 Authorization 헤더에 Bearer 토큰으로 전달을 해주면 됩니다

*/

                HttpHeaders naver_header = new HttpHeaders();
                naver_header.add("Authorization" , "Bearer " + naverOauthTokenDto.getAccess_token());
                HttpEntity<MultiValueMap<String , String>> naver_userInfoRequest_headers = new HttpEntity<>(naver_header);


/*

이 결과는 naver_userInfo_responseEntity 에 담기게 되고 마찬가지로 POJO 클래스로 변환을 해서 담아둡니다

"id" -> "Xqsn-5upLoB2DoT9qwaiOS-fSBg8TaBPArFdKLdXAsw"
"nickname" -> ""
"profile_image" -> "https://ssl.pstatic.net/static/pwe/address/img_profile.png"
"email" -> ""
"name" -> ""

그럼 이 결과는 다음과 같은 정보를 담고 있습니다 실제로 없는 값이 아니라 실명이 노출되어서 가렸습니다 그리고 특히 naver 는 id 라는 것을 던져주는데 이는 naver 에서 관리하는 유저의 유일한 key 값입니다

*/

               ResponseEntity<String> naver_userInfo_responseEntity = restTemplate.exchange(user_info , HttpMethod.GET , naver_userInfoRequest_headers , String.class);

                Map<String , Object> naver_convert_result = gson.fromJson(naver_userInfo_responseEntity.getBody() , Map.class);
                LinkedTreeMap naver_response = (LinkedTreeMap) naver_convert_result.get("response");
                NaverOauthUserInfoDto naverOauthUserInfoDto = new NaverOauthUserInfoDto(naver_response.get("id").toString() , naver_response.get("nickname").toString() , naver_response.get("email").toString() ,naver_response.get("name").toString());


/*

그 해당내용을 가지고 Authentication 객체를 만든다음 이를 우리의 JWT 를 생성하는 클래스로 집어넣어서 return 을 하게 됩니다

*/

               Authentication naver_authentication = new UsernamePasswordAuthenticationToken(naverOauthUserInfoDto.getEmail() ,
                        UUID.randomUUID()  ,
                        Arrays.asList(new SimpleGrantedAuthority("ROLE_MEMBER")));

                return  jwtGenerator.jwtGenerator(naver_authentication , javaWebKey);

        }

        return null;


    }

}


```

## NaverOauthTokenDto
```
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NaverOauthTokenDto {

    private String access_token;

    private String refresh_token;

    private String token_type;

    private int expires_in;

}


```

그리고 우리의 결과는 이를 access_token 과 refresh_token 으로 변환을 해서 클라이언트로 던져주게 됩니다 이건 이전 google 로그인때는 구현하지 않았는데 최근에 로직을 수정했습니다




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
    private RedisTemplate<String , String> redisTemplate;

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



        ValueOperations<String , String> jwt_token_key_value = redisTemplate.opsForValue();
        String refresh_redis_key = refresh_token;
        String refresh_redis_value = gson.toJson(authentication);
        Duration refresh_token_expireTime = Duration.ofMillis(refresh_tokenExpireTime);

        jwt_token_key_value.set(refresh_redis_key ,refresh_redis_value , refresh_token_expireTime);


        return result_token;

    }
}


```

아마 access_token 과 동일하지만 이전하고 달라진것이 있다면 지금 이곳에는 설명이 누락이 되었지만 사실 redis 연동으로 현재 refresh_token 을 준비할려고 합니다 이에 대한 거은 다음장에 다룰 예정입니다 일단은 기존에 access_token 만 return 을 해주는 것에서 이제는 access_token 과 refresh_token 을 같이 return 하는 것으로 변경이 되었습니다

## token_return
```

String to_json_token_result = gson.toJson(token);
to_json_token_result = Base64.getUrlEncoder().encodeToString(to_json_token_result.getBytes());


Cookie cookie = new Cookie("Oauth2_JWT_Token" , to_json_token_result);
cookie.setMaxAge(60);
cookie.setPath("/");
response.addCookie(cookie);

HttpHeaders headers = new HttpHeaders();
headers.add(HttpHeaders.SET_COOKIE , cookie.toString());

```

그런 다음 각 토큰을 이와 같이 return 을 해주면 됩니다 기존에는 단순 String 이었는데 이제는 key-value 형태로 return 해야 하니 Base64 인코딩으로 return 을 하고 있습니다

```

 let token_interval = setInterval(() => {

    const naver_jwt_token = getCookie("Oauth2_JWT_Token");
    
    if(naver_jwt_token){
        const decode_naver_jwt_token = commonJsonParser(commonUrlEncoder(naver_jwt_token));
        
        if(decode_naver_jwt_token.ACCESS_TOKEN && decode_naver_jwt_token.REFRESH_TOKEN){
            const project_name = f1().project_name;
            localStorage.setItem(`${project_name}_ACCESS_TOKEN` , decode_naver_jwt_token.ACCESS_TOKEN);
            localStorage.setItem(`${project_name}_REFRESH_TOKEN` , decode_naver_jwt_token.REFRESH_TOKEN);
            naver_oauth2_popup.close();
            clearInterval(token_interval)        
            setLogInVaild(true)
        }
    }

} , 1000)



```
토큰은 이와 같이 받아준다음 우리의 localStorage 에 저장을 해주시면됩니다


## commonJsonParser , commonUrlEncoder
```
export function commonUrlEncoder(str) {

    return decodeURIComponent(atob(str).split('').map(function (c) {
      return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
    }).join(''));

}

export function commonJsonParser(StringObject){


    return JSON.parse(StringObject);
}
 


```
단순 공통 함수입니다

## networkApiCall
```

export async function networkApiCall(api , method , param){

    const headers = new Headers();
    const project_name = f1().project_name;
   
    headers.append("Content-Type" , "application/json");
   

    const access_token = localStorage.getItem(`${project_name}_ACCESS_TOKEN`);
    const refresh_token = localStorage.getItem(`${project_name}_REFRESH_TOKEN`);

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
            window.location.href = '/login';
           
           
        }else {

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
그리고 다음번 요청때는 지금처럼 refresh_token 을 넣어주면됩니다

## CORSFilter

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CustomCorsFilter extends OncePerRequestFilter {

    @Value("${application.front.server}")
    private String front_server;
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        response.setHeader("Access-Control-Allow-Origin", front_server);
        response.setHeader("Access-Control-Allow-Credentials", "true");
        response.setHeader("Access-Control-Allow-Methods", "POST, GET, PUT, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers",
                "Origin, X-Requested-With, Content-Type, Accept, Authorization, x-xsrf-token , Authorization_Refresh" );

        // allow cros preflight
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
        } else {
            filterChain.doFilter(request, response);
        }
    }
}


```

새로운 헤더가 추가가 되었음으로 서버쪽 corsFilter 에 Authorization_Refresh 를 추가해줍니다 앞으로 refresh_token 은 이 key 값으로 헤더에 물려서 들어올 예정입니다