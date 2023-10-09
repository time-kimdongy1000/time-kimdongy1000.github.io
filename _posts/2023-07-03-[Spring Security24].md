---

title: Spring Secuirty 24 OAuth2 OAuth2 -  OAuth2AuthorizationRequestRedirectFilter
author: kimdongy1000
date: 2023-07-03 12:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

우리는 지난시간에 OAuth2LoginConfigurer 에 어떻게 설정이 되는지 살펴보는 와중에 OAuth2AuthorizationRequestRedirectFilter 객체가 만들어지는것을 확인했다 우리는 진짜 로그인을 하러 이제 들어갈것이다 시큐리티는 Filter 로 이루어진 프레임워크이기 떄문에 중요한 로직들은 전부 이 ~~Filter 로 끝이 나게 됩니다 그러면 천천히 들어가보겠습니다 

이 부분은 로그인 인증과 더불어서 승인코드를 발급받아서 시큐리티 안으로 들고 들어오는 역활을 하게 됩니다 

## 승인코드 
사실 승인코드라는 이름은 정식명칭이 아닌것으로 알고 있다 임시코드라고 하는 분도 있고 코드라고 말하는 사람도 있다 이 승인코드는 Authorization code grant 방식에서 

로그인을 하게 되면 인가서버는 코드를 하나 return 되는데 이 코드를 가지고 인가서버에서 access_token 을 발급받을때 이 코드를 쓰게 됩니다 

## 로그인 페이지 주소 

```
http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak

```

로그인 페이지 주소를 보게 되면 엄청 복잡한 로그인 주소가 쓰여 있는것을 볼 수 있습나다 이 로그인 주소는 어떻게 만들어지고 요청이 이루어지는지 알아보는데 그 핵심에 
OAuth2AuthorizationRequestRedirectFilter 가 존재합니다 



## OAuth2AuthorizationRequestRedirectFilter 

```
public class OAuth2AuthorizationRequestRedirectFilter extends OncePerRequestFilter {

    ...

    @Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		try {
			OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestResolver.resolve(request);
			if (authorizationRequest != null) {
				this.sendRedirectForAuthorization(request, response, authorizationRequest);
				return;
			}
		}
		catch (Exception ex) {
			this.unsuccessfulRedirectForAuthorization(request, response, ex);
			return;
		}
		try {
			filterChain.doFilter(request, response);
		}
		catch (IOException ex) {
			throw ex;
		}
		catch (Exception ex) {
			// Check to see if we need to handle ClientAuthorizationRequiredException
			Throwable[] causeChain = this.throwableAnalyzer.determineCauseChain(ex);
			ClientAuthorizationRequiredException authzEx = (ClientAuthorizationRequiredException) this.throwableAnalyzer
					.getFirstThrowableOfType(ClientAuthorizationRequiredException.class, causeChain);
			if (authzEx != null) {
				try {
					OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestResolver.resolve(request,
							authzEx.getClientRegistrationId());
					if (authorizationRequest == null) {
						throw authzEx;
					}
					this.sendRedirectForAuthorization(request, response, authorizationRequest);
					this.requestCache.saveRequest(request, response);
				}
				catch (Exception failed) {
					this.unsuccessfulRedirectForAuthorization(request, response, failed);
				}
				return;
			}
			if (ex instanceof ServletException) {
				throw (ServletException) ex;
			}
			if (ex instanceof RuntimeException) {
				throw (RuntimeException) ex;
			}
			throw new RuntimeException(ex);
		}
	}
}
```

필터는 한가지 특징이 무엇이냐면 무조건 doFilterInternal 를 재정의 해야 한다 이때 이 위주로 무조건 들어오기에 여기에 주석을 잡고 진행을 하겠습니다

그러면 우리는 이제 로그인을 해야 합니다 로그인을 하자마자 `OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestResolver.resolve(request);`
호출이 들어온것을 확인 할 수 있다 


## DefaultOAuth2AuthorizationRequestResolver

OAuth2AuthorizationRequest의 구현체는 기본적으로는 DefaultOAuth2AuthorizationRequestResolver 이 된다 여기서 
```

public final class DefaultOAuth2AuthorizationRequestResolver implements OAuth2AuthorizationRequestResolver {

    @Override
    public OAuth2AuthorizationRequest resolve(HttpServletRequest request) {
        String registrationId = resolveRegistrationId(request);
        if (registrationId == null) {
            return null;
        }
        String redirectUriAction = getAction(request, "login");
        return resolve(request, registrationId, redirectUriAction);
    }
}

```

`String registrationId = resolveRegistrationId(request);` 이 부분에서 registrationId 를 분리하게 되는데 filter 특정상 모든 요청이 거쳐가는것임으로 2번 정도 재호출될것이다 첫번째는 보통 파비콘이 호출됨으로 그 부분은 pass 하고 두번째 부분에서 registrationId 를 분리해해서 

registrationId 값은 keycloak 을 가져오게 된다 

`String redirectUriAction = getAction(request, "login");` 이 부분에서 다시 돌아올 수 있는 페이지를 객체로 만드는데 기본값은 login 이 된다 

그리고 다음코드에서 resolve 를 호출하게 되는데 


```

private OAuth2AuthorizationRequest resolve(HttpServletRequest request, String registrationId, String redirectUriAction) {
    if (registrationId == null) {
        return null;
    }
    ClientRegistration clientRegistration = this.clientRegistrationRepository.findByRegistrationId(registrationId);
    if (clientRegistration == null) {
        throw new IllegalArgumentException("Invalid Client Registration with Id: " + registrationId);
    }
    OAuth2AuthorizationRequest.Builder builder = getBuilder(clientRegistration);

    String redirectUriStr = expandRedirectUri(request, clientRegistration, redirectUriAction);

    // @formatter:off
    builder.clientId(clientRegistration.getClientId())
            .authorizationUri(clientRegistration.getProviderDetails().getAuthorizationUri())
            .redirectUri(redirectUriStr)
            .scopes(clientRegistration.getScopes())
            .state(DEFAULT_STATE_GENERATOR.generateKey());
    // @formatter:on

    this.authorizationRequestCustomizer.accept(builder);

    return builder.build();
    }

```

`ClientRegistration clientRegistration = this.clientRegistrationRepository.findByRegistrationId(registrationId);` 에서 저장된 ClientRegistration 을 가져오게 된다 이 값의 특징은 key 값으로 저장이 되고 key 값으로 불러올 수 있는데 여기서 key 값이 왜 keyClock 가 된것이냐면 

앞에 application.properties `spring.security.oauth2.client.registration.keycloak.clientId` 저장을 했을때 client 부분과 provider 부분이 모두 
keycloak 이 중간에 들어가게 되고 시큐리티는 이를 key 값으로 읽고 이를 저장하고 불러올 수 있게 repository 에 저장을 하게 된다 

왜 이렇게 저장되는지 기억이 안난다면 <https://time-kimdongy1000.github.io/posts/Spring-Security22/> 이 장에서 다루었으니 확인을 해보면 될것이다 

`OAuth2AuthorizationRequest.Builder builder = getBuilder(clientRegistration);` 이 부분에서 이제 인가서버에 무엇을 요청할지 저장을 하게 되는데 이 builder 값을 살펴보면 

```

builder = {OAuth2AuthorizationRequest$Builder@8104} 
 authorizationUri = null
 authorizationGrantType = {AuthorizationGrantType@8107} 
  value = "authorization_code"
 responseType = {OAuth2AuthorizationResponseType@8108} 
  value = "code"
 clientId = null
 redirectUri = null
 scopes = null
 state = null
 additionalParameters = {LinkedHashMap@8109}  size = 0
 parametersConsumer = {OAuth2AuthorizationRequest$Builder$lambda@8110} 
 attributes = {LinkedHashMap@8111}  size = 1
  "registration_id" -> "keycloak"
 authorizationRequestUri = null
 authorizationRequestUriFunction = {OAuth2AuthorizationRequest$Builder$lambda@8112} 
 uriBuilderFactory = {DefaultUriBuilderFactory@8113} 

```

대략 authorizationGrantType 방식은 authorization_code 방식이고 우리가 return 을 받을 데이터는 code 데이터가 된다 그리고 이런 정보를 조합해서 

`String redirectUriStr = expandRedirectUri(request, clientRegistration, redirectUriAction);` 를 만들어 내는데 

이 값을 살펴보면 http://localhost:8081/login/oauth2/code/keycloak 된다 이 부분은 우리가 제일 처음 keyClock 를 연동하고 admin 페이지에서 
redirect url 을 설정한것을 기억할것이다 다시 사진을 살펴보면

![9](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/5aee067e-2e1d-4439-a246-6db776467318)

이 부분에서 우리는 정확하게 이 Valid redirect URIs 를 설정했다 이 부분은 로그인 성공후 승인코드를 발급받고 나서 인가서버가 어디로 redirect 하는지 선언하게 된다 
참고로 인가서버에서 선언한 redirect url 과 시큐리티가 선언한 redirect url 이 다르면 에러가 발생한다 그럼 시큐리티는 승인코드를 가져와서 
http://localhost:8081/login/oauth2/code/keycloak  주소를 호출시키는 것이다 


```

builder.clientId(clientRegistration.getClientId())
            .authorizationUri(clientRegistration.getProviderDetails().getAuthorizationUri())
            .redirectUri(redirectUriStr)
            .scopes(clientRegistration.getScopes())
            .state(DEFAULT_STATE_GENERATOR.generateKey());

```

하단 builder 에서는 

```

builder = {OAuth2AuthorizationRequest$Builder@8104} 
 authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
 authorizationGrantType = {AuthorizationGrantType@8107} 
  value = "authorization_code"
 responseType = {OAuth2AuthorizationResponseType@8108} 
  value = "code"
 clientId = "Spring-Oauth2-Authorizaion-client"
 redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
 scopes = {Collections$UnmodifiableSet@8142}  size = 2
  0 = "email"
  1 = "profile"
 state = "KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4="
 additionalParameters = {LinkedHashMap@8109}  size = 0
 parametersConsumer = {OAuth2AuthorizationRequest$Builder$lambda@8110} 
 attributes = {LinkedHashMap@8111}  size = 1
 authorizationRequestUri = null
 authorizationRequestUriFunction = {OAuth2AuthorizationRequest$Builder$lambda@8112} 
 uriBuilderFactory = {DefaultUriBuilderFactory@8113} 

```

이런 정보가 담길것이다 요청하는 주소는 authorizationUri 이 될것이고 authorizationGrantType authorization_code 방식이고 retun 받고자 하는 데이터는 code 가 되는것이고 
이때 시큐리티가 원하는 정보는 email profile 가 되는것이고 

그리고 state 가 있는데 이에 대해서는 설명을 조금 하고 갈것이다 

## state

클라이언트가 임의의 값을 만들어서 code 요청에 담아서 보내고 인가서버는 이 값을 저장하고 있다가 다음 요청인 token 요청이 들어올때 이 클라이언트가 다시 한번 
state 값을 보내게 되는데 이 값이 code 요청때 값하고 비교해서 일치하면 access_token 이 발급이 되고 그렇지 않으면 거절이 되는 메커니즘을 가지게 됩니다 
그래서 이 state 는 다음장인 access_token 값을 요청할때 한번더 나오게 됩니다 


그럼 다시 돌아오면

```
OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestResolver.resolve(request);

if (authorizationRequest != null) {
    this.sendRedirectForAuthorization(request, response, authorizationRequest);
    return;
}

```

authorizationRequest 에 위해서 진행한 요청정보가 만들어지게 되고 시큐리티는 이 정보를 redirect 하게 됩니다 이 부분까지 시큐리티가 어떻게 로그인 페이지 요청을 만들어서 redirect 시키는지에 대해서 알아보았습니다 다음시간에는 진짜 로그인 을 통해서 승인코드 발급에 대해서 알아보겠습니다 







