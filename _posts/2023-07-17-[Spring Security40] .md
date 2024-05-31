---
title: Spring Secuirty 40 RestApi 기반 Google OAuth2 인증 만들기 1
author: kimdongy1000
date: 2023-07-17 14:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---


우리는 지난시간까지 우리가 등록한 User 로 Authentication 기반으로 JWT 토큰을 RSA 기반으로 만들어서 클라이언트에 던지고 다시 클라이언트에서 이 JWT 를 공개키로 복호화 해서 UserDetails 를 만들어서 인증을 하는 것까지 배웠습니다 오늘은 Google 의 Oauth2 인증으로 사용자 정보를 가져와서 저의 어플리케이션에 인증을 하는 방법에 대해서 알아보도록 하겠습니다 

## 구글 Api 콘솔 사이트 

https://console.cloud.google.com

이 부분에서 진행을 하게 됩니다 따로 특별히 설정하는 것은 다른 사이트에서 충분히 볼 수 있기에 저는 이 부분을 생략을 하고 바로 코드로 진입을 하겠습니다 
(구글에 검색하면 많이 나오기 때문에 생략했습니다 )

## Oauth2-client maven 추가 

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```
사실 Oauth2 라이브러리를 쓰지 않아도 충분히 구글 로그인 api 를 활용해서 통신을 할 수 있지만 저는 ClientRegistration 을 사용하기 위해서 이 라이브러리를 안고 가겠습니다 

## application.properties

```

spring:
  security:
    google_client_id : 
    google_client_secret : 
    google_redirect_url : http://localhost:8080/oauth2_google_login
    google_login_url: https://accounts.google.com/o/oauth2/auth
```

만약 oauth2-client 라이브러리를 사용한다면 이 정도 정보만 가지고 있으면 된다 그게 아니라면 authorizationUri , tokenUri , userInfoUri 는 필수값인데 만약 사용하지 않는다는 예제는 다음 포스트 네이버 Oauth2 인증을 하면서 알아보도록 하겠습니다 Google 인증은 이 정도 정보만 있어도 사용이 가능합니다 

## Oauth2Config

```
@Configuration
public class Oauth2Config {

    @Value("${spring.security.google_client_id}")
    private String google_client_id;

    @Value("${spring.security.google_client_secret}")
    private String google_client_secret;

    @Value("${spring.security.google_redirect_url}")
    private String google_redirect_url;


    private ClientRegistration googleClientRegistration(){
        return CommonOAuth2Provider.GOOGLE.getBuilder("google")
                .clientId(google_client_id)
                .clientSecret(google_client_secret)
                .redirectUri(google_redirect_url)
                .build();
    }

    
    @Bean
    public ClientRegistrationRepository clientRegistrationRepository(){
        ClientRegistrationRepository clientRegistrationRepository = new InMemoryClientRegistrationRepository(googleClientRegistration());
        return clientRegistrationRepository;
    }
}


```

먼저 config 를 만들것인데 이 부분은 ClientRegistration 을 관리하기 위한 Bean 설정을 위한 config 입니다 이때 Google 같은 대표적인 인증 사이트는 CommonOAuth2Provider 가 제공이 되어 이때는 각 어플리케이션에 제공이 되어야 하는 
client_id , client_secret , redirect_url 만 넣어주면 됩니다 그럼 잠깐 CommonOAuth2Provider 살펴보면

## CommonOAuth2Provider
```
public enum CommonOAuth2Provider {

    GOOGLE {
        public ClientRegistration.Builder getBuilder(String registrationId) {
            ClientRegistration.Builder builder = this.getBuilder(registrationId, ClientAuthenticationMethod.CLIENT_SECRET_BASIC, "{baseUrl}/{action}/oauth2/code/{registrationId}");
            builder.scope(new String[]{"openid", "profile", "email"});
            builder.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth");
            builder.tokenUri("https://www.googleapis.com/oauth2/v4/token");
            builder.jwkSetUri("https://www.googleapis.com/oauth2/v3/certs");
            builder.issuerUri("https://accounts.google.com");
            builder.userInfoUri("https://www.googleapis.com/oauth2/v3/userinfo");
            builder.userNameAttributeName("sub");
            builder.clientName("Google");
            return builder;
        }
    }
}

```
이렇게 google 같은 경우는 기본적으로 이렇게 미리 설정이 되어 있습니다 우리는 이 정보를 그대로 가져가서 진행을 하겠습니다 그럼 설정은 끝났습니다 

## React 에서 구글 로그인 버튼 클릭 

```

 const oauth2_google_login = (event) => {

    const api = "/oauth2/google/oauth2_login_address"
    const method= "get"
    const param = {}

    networkApiCall(api , method , param).then(result => {

        
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
이 구글 로그인 버튼을 클릭하게 되면 back-end 에서 인증을 할 수 있는 url 주소를 만들어서 client 로 return 을 하게 됩니다 예를 들면 이렇게 말이죠 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/8e599292-01f1-4c02-84a7-4524534c10d3)

지금 화면에서 구글 로그인 버튼을 클릭하면 옆에 팝업으로 뜨는게 현재의 상태입니다 그럼 이때 url 을 가져오는 부분을 back-end 로 보게 되면

```
@RestController
public class LoginController {

    @Autowired
    private LoginService loginService;

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
                case "google" :

                    clientRegistration = clientRegistrationRepository.findByRegistrationId("google");
                    Set<String> scopes = clientRegistration.getScopes();
                    String string_scope = String.join(" ", clientRegistration.getScopes());
                    return_url.append(clientRegistration.getProviderDetails().getAuthorizationUri());
                    return_url.append("?");
                    return_url.append("client_id=");
                    return_url.append(clientRegistration.getClientId());
                    return_url.append("&");
                    return_url.append("redirect_uri=");
                    return_url.append(clientRegistration.getRedirectUri());
                    return_url.append("&");
                    return_url.append("response_type=code&");
                    return_url.append("scope=");
                    return_url.append(string_scope);

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


```

이 부분이 현재 Google 의 저 팝업을 띄우기 위한 url 주소를 만드는 부분입니다 그러면 구글 api 를 읽어서 파라미터를 만든뒤 다시 client 로 return 하게 되면 

```

networkApiCall(api , method , param).then(result => {
        
    const options = 'width=700, height=600, top=50, left=50, scrollbars=yes popup=true';
    const google_oauth2_popup = window.open(result.Oauth2_login_url ,'popup' ,  options);
})



```
중간에 생햑하고 result 에 key 값이 Oauth2_login_url 로 넘어오게 됩니다 그러면 사용자는 저의 어플리케이션 사이트에 접속을 해서 인증을 하는게 아니라 이때 만큼은 google 에 요청을 해서 
인증을 받게 됩니다 그러면 그 인증이 완료 되고 난 다음에는 다음장에서 정리하도록 하겠습니다 

