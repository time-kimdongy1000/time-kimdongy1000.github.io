---
title: Spring Secuirty 41 RestApi 기반 Google OAuth2 인증 만들기 2
author: kimdongy1000
date: 2023-07-17 15:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

지난시간에는 구글 로그인 버튼을 클릭시 구글에서 제공하는 Oauth2 로그인을 시도하게 됩니다 이때 우리가 로그인을 완료하면 구글 서버는 제가 등록한 redirect 주소로 code 값을 던저주게 됩니다 우리는 이 code 로 access_token 을 요청해서 받는 진행을 하겠습니다

## LoginController

```
@RestController
public class LoginController {

    @Autowired
    private LoginService loginService;

    @Autowired
    private Gson gson;

    @Autowired
    private ClientRegistrationRepository clientRegistrationRepository;



    @GetMapping("/oauth2_{oauth2_service}_login")
    public ResponseEntity<?> google_login(
            @RequestParam("code") String code ,
            @PathVariable("oauth2_service") String oauth2_service,
            HttpServletResponse response

        )
    {
        try{

            String jwtToken = loginService.oauth2_loadByUser(code , oauth2_service);

            Map<String , Object> result = new HashMap<>();
            String to_json_result = gson.toJson(result);

            Cookie cookie = new Cookie("Oauth2_JWT_Token" , jwtToken);
            cookie.setMaxAge(60);
            cookie.setPath("/");
            response.addCookie(cookie);

            HttpHeaders headers = new HttpHeaders();
            headers.add(HttpHeaders.SET_COOKIE , cookie.toString());


            return new ResponseEntity<>(to_json_result ,headers ,HttpStatus.OK);
        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}

```

이때 제가 등록한 redirect_url 은 http://localhost:8080/oauth2_google_login 이렇게 들어오게 됩니다 저는 이를 동적으로 받을 수 있게 @PathVariable 로 받게끔 설정을 했습니다 저는 여기에 naver , 애플 로그인을 붙이는 거 까지 진행을 할것입니다그러면 구글 서버에서 온 code 를 가져오게 되면서 `String jwtToken = loginService.oauth2_loadByUser(code , oauth2_service);` 이 부분에서 서버와 통신을 하게 됩니다

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



    public String oauth2_loadByUser(String code , String oauth_service) throws Exception{


        ClientRegistration clientRegistration = null;


        String client_id = null;
        String client_secret = null;
        String redirect_url = null;
        String token_url = null;
        String user_info = null;

        RestTemplate restTemplate = new RestTemplate();
        Map<String , Object> params = new HashMap<>();


        switch (oauth_service) {
            case "google":

        /*

        이때 우리가 앞에서 사용한 oauth2-client 를 사용하기 위함이 나옵니다 이때 clientRegistration 저장된 값들을 가져올것입니다
        그러면 메모리에 저장된 clientRegistration 에서 key 값이 google 인 값을 가져오게 됩니다

        */
        clientRegistration = clientRegistrationRepository.findByRegistrationId("google");

        /*

        그래서 access_token 을 가져오기 위해서
        1. 클라이언트 아이디
        2. 클라이언트 시크릿
        3. 리다이렉트 주소
        4. 토큰 요청주소 (이때 토큰은 access_token 입니다 )
        5. access_token 으로 user 정보를 가져올 user_info 주소를 가져오게 됩니다

        */

        client_id = clientRegistration.getClientId();
        client_secret = clientRegistration.getClientSecret();
        redirect_url = clientRegistration.getRedirectUri();
        token_url = clientRegistration.getProviderDetails().getTokenUri();
        user_info = clientRegistration.getProviderDetails().getUserInfoEndpoint().getUri();


        params.put("code" , code);
        params.put("client_id" , client_id);
        params.put("client_secret" , client_secret);
        params.put("redirect_uri" , redirect_url);
        params.put("grant_type" , "authorization_code");

    /*

        서버로의 요청이 들어갑니다
        그럼 이 google_access_token_responseEntity 의 결과는 다음과 같이 나오게 됩니다

        {
            
        "access_token": "ya29.a0AXooCguX8ognkNE6bRP1Fkj1E_rZ3X8vRTUUg59aZtmoZi13WJjBxfXO_p_YPNRDVaTvlsKnlFNTIsCqXTNjyyADgJH0e3hM7vVoujdNy1llVoh4NAlsbNjTCW26XYTBY3ZcCPoTG_QfyJWdjFnwG5eqZ1KLVskOWUBJaCgYKAawSARESFQHGX2Miu5y0mbmH5D2kCEqlWvr6wQ0171",
        "expires_in": 3599,
        "scope": "https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile openid",
        "token_type": "Bearer",
        "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjMyM2IyMTRhZTY5NzVhMGYwMzRlYTc3MzU0ZGMwYzI1ZDAzNjQyZGMiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJhenAiOiIzNDkzMjMyODA4MTgtNHB1YXVqam03ZTVjaWloMGUzbGpkY3YxdnRoajRob3UuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJhdWQiOiIzNDkzMjMyODA4MTgtNHB1YXVqam03ZTVjaWloMGUzbGpkY3YxdnRoajRob3UuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJzdWIiOiIxMTM5NzcxODIzMjg3OTQzNzYyOTEiLCJlbWFpbCI6ImtpbWRvbmd5MTAwMEBnbWFpbC5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXRfaGFzaCI6Im9PODA4bXZ3Q2lIX0RxeFRUYmZqVEEiLCJuYW1lIjoi7JuU6riJ7Jik64qYICjsmKTripgg7JuU6riJKSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS9BQ2c4b2NMb3YtblFKNjFIZWM1QlZEUlJMLVdCaUpyNVROYVY4UjRXalhDVVBjWWVpRHBXVHQtcD1zOTYtYyIsImdpdmVuX25hbWUiOiLsmKTripgiLCJmYW1pbHlfbmFtZSI6IuyblOq4iSIsImlhdCI6MTcxNjQyNjUwNiwiZXhwIjoxNzE2NDMwMTA2fQ.AWRjwV6w_pGD3UbuDyv91EAizHgv6ZRbAyyhaS0PdL9NQmcg5k20MVD0hFJeuyFgGTDtOsvASKjo_wTXuqmVE97fqV4NyHS-8n-rkEF3g9hyQMKMjWpeiqwx1xdOcp2lLH3frX5XRN48WUhXmVMiJQtDCW7icww4ooeTKlvO5bM6qboIm_fABOncPq8zIsx9wd7i3GWEYpStLOYBd7sm5cyIDyulukkHFwQZ87bf5bnwsaoSJlINg-_5mvzT5tCZMZBk7yRpzKb1ydV0qlfHgC49OwG3ZrM2n2V2v1kej7AcrYDv5znCT_xS501t98zSnY430NvG0D8S6smyIoQJfQ"

        }

        자 여기서는 access_token 과 id_token 이 있습니다 기본적으로 구글은 openid connect 를 가지고 있기 때문에 기본적으로 scope 에 openid 이 있으면 openid 토큰이 있는것은 기본적으로 id_token 을 제공합니다 이를 통해서 id_token 만 넘어오는 것만으로도 인증이 완료된것입니다 그래서 이 id_token 을 jwt.io 가져다 해석을 하게 되면

        {

        "iss": "https://accounts.google.com",
        "azp": "349323280818-4puaujjm7e5ciih0e3ljdcv1vthj4hou.apps.googleusercontent.com",
        "aud": "349323280818-4puaujjm7e5ciih0e3ljdcv1vthj4hou.apps.googleusercontent.com",
        "sub": "113977182328794376291",
        "email": "kimdongy1000@gmail.com",
        "email_verified": true,
        "at_hash": "oO808mvwCiH_DqxTTbfjTA",
        "name": "월급오늘 (오늘 월급)",
        "picture": "https://lh3.googleusercontent.com/a/ACg8ocLov-nQJ61Hec5BVDRRL-WBiJr5TNaV8R4WjXCUPcYeiDpWTt-p=s96-c",
        "given_name": "오늘",
        "family_name": "월급",
        "iat": 1716426506,
        "exp": 1716430106
        
        }

        이렇게 나오게 됩니다 이것들이 제가 설정한 oauth2 의 정보입니다

    */

    ResponseEntity<String> google_access_token_responseEntity = restTemplate.postForEntity(token_url , params , String.class);

    if(google_access_token_responseEntity.getStatusCode() != HttpStatus.OK){
        throw new RuntimeException("구글 로그인에 실패했습니다.");
    }


    /*

    우리는 id_token 을 제외하고도 access_token 을 이용해서 user 정보를 가져오겠습니다 그러면 우리는 요청받은 정보를 바탕으로 Spring POJO 객체를 만들어서 변환을 합니다

    이제 이 access_token 을 이용해서 헤더에 Authorization key Bearer 에 access_token 을 넣고 요청을 보내게 됩니다


    */

    GoogleOauthTokenDto googleOauthTokenDto = gson.fromJson(google_access_token_responseEntity.getBody() , GoogleOauthTokenDto.class);

    HttpHeaders google_headers = new HttpHeaders();
    google_headers.add("Authorization" , "Bearer " + googleOauthTokenDto.getAccess_token());
    HttpEntity<MultiValueMap<String , String>> google_userInfoRequest_headers = new HttpEntity<>(google_headers);
           
    /*

    그럼 이곳에서 우리가 access_token 으로 요청한 Oauth2 구글 정보를 가져옵니다

    {

        "sub": "113977182328794376291",
        "name": "월급오늘 (오늘 월급)",
        "given_name": "오늘",
        "family_name": "월급",
        "picture": "https://lh3.googleusercontent.com/a/ACg8ocLov-nQJ61Hec5BVDRRL-WBiJr5TNaV8R4WjXCUPcYeiDpWTt-p\u003ds96-c",
        "email": "kimdongy1000@gmail.com",
        "email_verified": true,
        "locale": "ko"
    }


    그럼 이렇게 나옵니다 아까 id_token 하고는 조금 다른 정보가 나오는데 이게 id_token 과 access_token 으로 user_info 정보와는 다르게 됩니다
*/

    ResponseEntity<String> google_userInfo_responseEntity = restTemplate.exchange(user_info , HttpMethod.GET , google_userInfoRequest_headers , String.class);

/*

    마찬가지로 이를 바탕으로 다시 Spring POJO 객체로 변환을 해서 return 하면 끝입니다 이곳으로 구글하고의 주고받는것은 끝입니다 그런데 밑은 무엇이냐
    실제로 위의 정보를 바로 사용해도 되지만 저는 이 정보를 바탕으로 다시 저만의 JWT 객체로 만들어서 클라이언트로 return 하게 됩니다 그 부분이 하단에 나타납니다

*/
    GoogleOauthUserInfoDto googleOauthUserInfoDto = gson.fromJson(google_userInfo_responseEntity.getBody() , GoogleOauthUserInfoDto.class);


/*

    그래서 구글에서 준 정보를 바탕으로 Authentication 객체를 만들어서 jwtGenerator 로 return 하게 됩니다
    만들어지는 과정은 지난시간 참고 하시면됩니다

*/
        Authentication google_authentication = new UsernamePasswordAuthenticationToken( googleOauthUserInfoDto.getEmail() ,
                UUID.randomUUID()  ,
                Arrays.asList(new SimpleGrantedAuthority("ROLE_MEMBER")));

        return  jwtGenerator.jwtGenerator(google_authentication , javaWebKey);

    }

    return null;

    }

}



```


## GoogleOauthTokenDto

```
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class GoogleOauthTokenDto {

    private String access_token;
    private int expires_in;
    private String scope;
    private String token_type;
    private String id_token;

}


```

## GoogleOauthUserInfoDto

```
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class GoogleOauthUserInfoDto {

    private String id;
    private String email;
    private boolean verified_email;

    private String name;

    private String given_name;

    private String picture;

    private String locale;
}


```

## 제가 만든 토큰으로 return
```

String jwtToken = loginService.oauth2_loadByUser(code , oauth2_service);

Map<String , Object> result = new HashMap<>();
String to_json_result = gson.toJson(result);


Cookie cookie = new Cookie("Oauth2_JWT_Token" , jwtToken);
cookie.setMaxAge(60);
cookie.setPath("/");
response.addCookie(cookie);

HttpHeaders headers = new HttpHeaders();
headers.add(HttpHeaders.SET_COOKIE , cookie.toString());


return new ResponseEntity<>(to_json_result ,headers ,HttpStatus.OK);

```

그러면 받은 정보를 바탕으로 JWT 토큰을 만들어서 클라이언트에 쿠키 형태로 response 하면끝이 납니다

## 클라이언트에서 토큰 저장

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

            const project_name = f1().project_name;
            localStorage.setItem(`${project_name}_ACCESS_TOKEN` , google_jwt_token);
            google_oauth2_popup.close();
            clearInterval(token_interval)        
            setLogInVaild(true)
        }

    } , 1000)

}).catch(error => {

    console.log(error)
    
})

}

```
그러면 토큰은 팝업형태로 넘어와서 저의 localStorage 저장을 하고 팝업을 자동으로 닫는 로직을 완성하면 우리는 Google 로그인을 이용해서 만들게 된것입니다 


