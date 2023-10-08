---

title: Spring Secuirty 23 OAuth2 OAuth2-OAuth2LoginConfigurer 
author: kimdongy1000
date: 2023-07-02 12:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

우리는 지난시간에 keyClock 와 Spring - Security 를 연동할때 사용한 ClientRegistration 에 대해서 알아보았습니다 이번시간에는 실제 로그인이 일어나는 과정에 대해서 알아보도록 하겠습니다 


## OAuth2LoginConfigurer

기본적으로 이 페이지를 열어보면 소스가 무지 많다 다만 Oauth2 와 관련된 소스들의 특징이 있는데 Configurer 시큐리티만의 바로 최상단 인터페이스가 하나 존재하는데 

`SecurityConfigurer.interface` 여기를 들어가보면 

```
package org.springframework.security.config.annotation;


public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {

	void init(B builder) throws Exception;


	void configure(B builder) throws Exception;

}


```

이렇게 2개의 메서드가 시그니처만 정의가 되어 있다 즉 최상단 인터페이스로 SecurityConfigurer 상속받고 있으면 하단 클래스단에서는 init , configure 구현을 해주어야 한다 
그렇기에 `OAuth2LoginConfigurer.java` 도 최상단 인터페이스로 상속을 받기에 init , configure 재정의 하고 있으며 이 중에서는 init 위주로 보면됩니다 



## OAuth2LoginConfigurer init
```

@Override
public void init(B http) throws Exception {
    OAuth2LoginAuthenticationFilter authenticationFilter = new OAuth2LoginAuthenticationFilter(
            OAuth2ClientConfigurerUtils.getClientRegistrationRepository(this.getBuilder()),
            OAuth2ClientConfigurerUtils.getAuthorizedClientRepository(this.getBuilder()), this.loginProcessingUrl);
    this.setAuthenticationFilter(authenticationFilter);
    super.loginProcessingUrl(this.loginProcessingUrl);
    if (this.loginPage != null) {
        // Set custom login page
        super.loginPage(this.loginPage);
        super.init(http);
    }
    else {
        Map<String, String> loginUrlToClientName = this.getLoginLinks();
        if (loginUrlToClientName.size() == 1) {
            // Setup auto-redirect to provider login page
            // when only 1 client is configured
            this.updateAuthenticationDefaults();
            this.updateAccessDefaults(http);
            String providerLoginPage = loginUrlToClientName.keySet().iterator().next();
            this.registerAuthenticationEntryPoint(http, this.getLoginEntryPoint(http, providerLoginPage));
        }
        else {
            super.init(http);
        }
    }
    OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest> accessTokenResponseClient = this.tokenEndpointConfig.accessTokenResponseClient;
    if (accessTokenResponseClient == null) {
        accessTokenResponseClient = new DefaultAuthorizationCodeTokenResponseClient();
    }
    OAuth2UserService<OAuth2UserRequest, OAuth2User> oauth2UserService = getOAuth2UserService();
    OAuth2LoginAuthenticationProvider oauth2LoginAuthenticationProvider = new OAuth2LoginAuthenticationProvider(
            accessTokenResponseClient, oauth2UserService);
    GrantedAuthoritiesMapper userAuthoritiesMapper = this.getGrantedAuthoritiesMapper();
    if (userAuthoritiesMapper != null) {
        oauth2LoginAuthenticationProvider.setAuthoritiesMapper(userAuthoritiesMapper);
    }
    http.authenticationProvider(this.postProcess(oauth2LoginAuthenticationProvider));
    boolean oidcAuthenticationProviderEnabled = ClassUtils
            .isPresent("org.springframework.security.oauth2.jwt.JwtDecoder", this.getClass().getClassLoader());
    if (oidcAuthenticationProviderEnabled) {
        OAuth2UserService<OidcUserRequest, OidcUser> oidcUserService = getOidcUserService();
        OidcAuthorizationCodeAuthenticationProvider oidcAuthorizationCodeAuthenticationProvider = new OidcAuthorizationCodeAuthenticationProvider(
                accessTokenResponseClient, oidcUserService);
        JwtDecoderFactory<ClientRegistration> jwtDecoderFactory = this.getJwtDecoderFactoryBean();
        if (jwtDecoderFactory != null) {
            oidcAuthorizationCodeAuthenticationProvider.setJwtDecoderFactory(jwtDecoderFactory);
        }
        if (userAuthoritiesMapper != null) {
            oidcAuthorizationCodeAuthenticationProvider.setAuthoritiesMapper(userAuthoritiesMapper);
        }
        http.authenticationProvider(this.postProcess(oidcAuthorizationCodeAuthenticationProvider));
    }
    else {
        http.authenticationProvider(new OidcAuthenticationRequestChecker());
    }
    this.initDefaultLoginFilter(http);
}



```

우리는 여기에 주석을 걸고 진행을 할 예정입니다 결국 이 파일은 기동을 할때 KeyClock 의 기본적인 설정을 하는곳입니다 
아니 앞에도 기본설정이라면서 뭔 또 기본설정이야 할 수 있지만 실제로 앞에는 연동을 위한 최소한의 정보를 기입해서 ClientRegistration 에 넣었다면 이제는 여기에 들어간 정보를 가지고 기본적인 로그인 페이지 , 인증이 안되었으면 되돌아와야 하는 페이지 등 보다 구체적인 설정을 하는곳입니다 


## OAuth2LoginAuthenticationFilter
```

OAuth2LoginAuthenticationFilter authenticationFilter = new OAuth2LoginAuthenticationFilter(
				OAuth2ClientConfigurerUtils.getClientRegistrationRepository(this.getBuilder()),
				OAuth2ClientConfigurerUtils.getAuthorizedClientRepository(this.getBuilder()), this.loginProcessingUrl);

```

OAuth2LoginAuthenticationFilter 는 Oauth2Login 의 시작에 달려 있는 Filter 입니다 시큐리티는 Filter 로 이어져 있는 프레임워크로 OAuth2LoginAuthenticationFilter
가 사용자의 계정정보를 받아서 인증이 완료되면 다시 되돌아오면서 승인코드를 같이 가지고 와주는 객체입니다 그래서 OAuth2 에서는 제일 먼저 이 Filter 에 대한 기본적인 정보를 세팅을 해줍니다 

OAuth2LoginAuthenticationFilter 의 객체를 만드는데 두가지 파라미터를 받고 있습니다 

1. OAuth2ClientConfigurerUtils.getClientRegistrationRepository(this.getBuilder())

2. OAuth2ClientConfigurerUtils.getAuthorizedClientRepository(this.getBuilder())

첫번째 파라미터는 우리가 앞에서 설정한 OAuth2ClientRegistrationRepositoryConfiguration 에서 등록한 ClientRegistrationRepository 에 등록된 정보를 가져와서 세팅을 하게 됩니다 

두번째 파라미터는 메서드 명에서 알 수 있다 싶히 getAuthorizedClientRepository 이미 인증된 ClientRegirationRepository 를 가져오는 곳인데 최초 런타임시에는 이곳에 값이 없기 때문에 OAuth2ClientConfigurerUtils.getAuthorizedClientRepository(this.getBuilder()) 아무 값도 들어오지 않습니다 

최종적으로 이 루트가 끝이 나게 되면 시큐리티는 
ClientRegistrationRepository ->  AuthorizedClientRepository 로 이동을 시키게 됩니다 

만들어진 authenticationFilter 객체의 값을 보게 되면 

```

authenticationFilter = {OAuth2LoginAuthenticationFilter@5432} 

 clientRegistrationRepository = {InMemoryClientRegistrationRepository@5433} 
  registrations = {Collections$UnmodifiableMap@5450}  size = 1
   "keycloak" -> {ClientRegistration@5455} "ClientRegistration{registrationId='keycloak', clientId='Spring-Oauth2-Authorizaion-client', clientSecret='NIe2qftuPcclGWFiBFicEWoK5SfYs7ql', clientAuthenticationMethod=org.springframework.security.oauth2.core.ClientAuthenticationMethod@86baaa5b, authorizationGrantType=org.springframework.security.oauth2.core.AuthorizationGrantType@5da5e9f3, redirectUri='http://localhost:8081/login/oauth2/code/keycloak', scopes=[email, profile], providerDetails=org.springframework.security.oauth2.client.registration.ClientRegistration$ProviderDetails@3b1dc579, clientName='Spring-Oauth2-Authorizaion-client'}"
    key = "keycloak"
    value = {ClientRegistration@5455} "ClientRegistration{registrationId='keycloak', clientId='Spring-Oauth2-Authorizaion-client', clientSecret='NIe2qftuPcclGWFiBFicEWoK5SfYs7ql', clientAuthenticationMethod=org.springframework.security.oauth2.core.ClientAuthenticationMethod@86baaa5b, authorizationGrantType=org.springframework.security.oauth2.core.AuthorizationGrantType@5da5e9f3, redirectUri='http://localhost:8081/login/oauth2/code/keycloak', scopes=[email, profile], providerDetails=org.springframework.security.oauth2.client.registration.ClientRegistration$ProviderDetails@3b1dc579, clientName='Spring-Oauth2-Authorizaion-client'}"
     registrationId = "keycloak"
     clientId = "Spring-Oauth2-Authorizaion-client"
     clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
     clientAuthenticationMethod = {ClientAuthenticationMethod@5460} 
     authorizationGrantType = {AuthorizationGrantType@5461} 
     redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
     scopes = {Collections$UnmodifiableSet@5463}  size = 2
     providerDetails = {ClientRegistration$ProviderDetails@5464} 
     clientName = "Spring-Oauth2-Authorizaion-client"

 authorizedClientRepository = {AuthenticatedPrincipalOAuth2AuthorizedClientRepository@5434} 
  authenticationTrustResolver = {AuthenticationTrustResolverImpl@5473} 
   anonymousClass = {Class@5477} "class org.springframework.security.authentication.AnonymousAuthenticationToken"
    cachedConstructor = null
    newInstanceCallerCache = null
    name = "org.springframework.security.authentication.AnonymousAuthenticationToken"
    module = {Module@6736} "unnamed module @6a714237"
    classLoader = {ClassLoaders$AppClassLoader@5078} 
    packageName = "org.springframework.security.authentication"
    componentType = {int[0]@6738} []
    reflectionData = null
    classRedefinedCount = 0
    genericInfo = null
    enumConstants = null
    enumConstantDirectory = null
    annotationData = null
    annotationType = null
    classValueMap = null
   rememberMeClass = {Class@5478} "class org.springframework.security.authentication.RememberMeAuthenticationToken"
  authorizedClientService = {InMemoryOAuth2AuthorizedClientService@5474} 
  anonymousAuthorizedClientRepository = {HttpSessionOAuth2AuthorizedClientRepository@5475} 

```

좀보기 힘들겠지만 key 값으로 보면 clientRegistrationRepository 에는 우리가 앞에서 설정한 issur 에 따라서 세팅된 반면 authorizedClientRepository 아무것도 없는것을 볼 수 있습니다


```

this.setAuthenticationFilter(authenticationFilter);
super.loginProcessingUrl(this.loginProcessingUrl); 

```

그리고 두줄만 따로보면 위에서 만들어진 authenticationFilter 객체를 세팅을 하고 로그인 processingUrl 을 설정하게 되는데 시큐리티 기본값인 /login/oauth2/code/*
이 값을 기본값으로 두고 있습니다 즉 /login/oauth2/code/* authenticationFilter 를 동작을 시키겠다는 뜻입니다 


```

if (this.loginPage != null) {
    // Set custom login page
    super.loginPage(this.loginPage);
    super.init(http);
}

```

이 부분은 제공하는 로그인페이지가 있는지입니다 이때는 제공하는게 인가서버 측이 아니라 Client 측 기준입니다 우리는 따로 만들지 않고 KeyClock 가 만들어준 페이지를 사용할것임으로 이 소스는 넘어가게 됩니다 

```

Map<String, String> loginUrlToClientName = this.getLoginLinks();
if (loginUrlToClientName.size() == 1) {
    // Setup auto-redirect to provider login page
    // when only 1 client is configured
    this.updateAuthenticationDefaults();
    this.updateAccessDefaults(http);
    String providerLoginPage = loginUrlToClientName.keySet().iterator().next();
    this.registerAuthenticationEntryPoint(http, this.getLoginEntryPoint(http, providerLoginPage));
    }

```

이 하단은 그러면 Client 가 제공하는 로그인 페이지는 없기 때문에 인가서버가 제공하는 로그인 페이지를 세팅하는 곳입니다 

`String providerLoginPage = loginUrlToClientName.keySet().iterator().next();` 의 값은 /oauth2/authorization/keycloak 이 로그인 페이지가 되게 됩니다 

`this.registerAuthenticationEntryPoint(http, this.getLoginEntryPoint(http, providerLoginPage));`
그리고 마지막 줄은 인가가 되지 않았을때 클라이언트를 다시 로그인 페이지로 리디렉션을 시키는 소스입니다 


```
private AuthenticationEntryPoint getLoginEntryPoint(B http, String providerLoginPage) {
    RequestMatcher loginPageMatcher = new AntPathRequestMatcher(this.getLoginPage());
    RequestMatcher faviconMatcher = new AntPathRequestMatcher("/favicon.ico");
    RequestMatcher defaultEntryPointMatcher = this.getAuthenticationEntryPointMatcher(http);
    RequestMatcher defaultLoginPageMatcher = new AndRequestMatcher(
            new OrRequestMatcher(loginPageMatcher, faviconMatcher), defaultEntryPointMatcher);
    RequestMatcher notXRequestedWith = new NegatedRequestMatcher(
            new RequestHeaderRequestMatcher("X-Requested-With", "XMLHttpRequest"));
    LinkedHashMap<RequestMatcher, AuthenticationEntryPoint> entryPoints = new LinkedHashMap<>();
    entryPoints.put(new AndRequestMatcher(notXRequestedWith, new NegatedRequestMatcher(defaultLoginPageMatcher)),
            new LoginUrlAuthenticationEntryPoint(providerLoginPage));
    DelegatingAuthenticationEntryPoint loginEntryPoint = new DelegatingAuthenticationEntryPoint(entryPoints);
    loginEntryPoint.setDefaultEntryPoint(this.getAuthenticationEntryPoint());
    return loginEntryPoint;
}


```

위에서도 말했지만 이 페이지는 기본적인 로그인 페이지를 세팅함과 동시에 권한이 없거나 상실한 사람이 다시 인증을 받으로 올때를 대비해서 시큐리티에서 세팅을 해두는 곳입니다 

```
DelegatingAuthenticationEntryPoint loginEntryPoint = new DelegatingAuthenticationEntryPoint(entryPoints);
loginEntryPoint.setDefaultEntryPoint(this.getAuthenticationEntryPoint());

```

결국 이 두줄이 핵심인데 loginEntryPoint 아래와 같이 값을 가지게 됩니다 

```

loginEntryPoint = {DelegatingAuthenticationEntryPoint@6801} 
 entryPoints = {LinkedHashMap@6795}  size = 1
  {AndRequestMatcher@6807} "And [Not [RequestHeaderRequestMatcher [expectedHeaderName=X-Requested-With, expectedHeaderValue=XMLHttpRequest]], Not [And [Or [Ant [pattern='/login'], Ant [pattern='/favicon.ico']], And [Not [RequestHeaderRequestMatcher [expectedHeaderName=X-Requested-With, expectedHeaderValue=XMLHttpRequest]], MediaTypeRequestMatcher [contentNegotiationStrategy=org.springframework.web.accept.HeaderContentNegotiationStrategy@777d191f, matchingMediaTypes=[application/xhtml+xml, image/*, text/html, text/plain], useEquals=false, ignoredMediaTypes=[*/*]]]]]]" -> {LoginUrlAuthenticationEntryPoint@6808} 
   key = {AndRequestMatcher@6807} "And [Not [RequestHeaderRequestMatcher [expectedHeaderName=X-Requested-With, expectedHeaderValue=XMLHttpRequest]], Not [And [Or [Ant [pattern='/login'], Ant [pattern='/favicon.ico']], And [Not [RequestHeaderRequestMatcher [expectedHeaderName=X-Requested-With, expectedHeaderValue=XMLHttpRequest]], MediaTypeRequestMatcher [contentNegotiationStrategy=org.springframework.web.accept.HeaderContentNegotiationStrategy@777d191f, matchingMediaTypes=[application/xhtml+xml, image/*, text/html, text/plain], useEquals=false, ignoredMediaTypes=[*/*]]]]]]"
   value = {LoginUrlAuthenticationEntryPoint@6808} 
    portMapper = {PortMapperImpl@6811} 
    portResolver = {PortResolverImpl@6812} 
    loginFormUrl = "/oauth2/authorization/keycloak"
    forceHttps = false
    useForward = false
    redirectStrategy = {DefaultRedirectStrategy@6813} 
 defaultEntryPoint = null

```

객체의 값이 많지만 결국은 이 뜻입니다 시큐리티가 인증 인가가 없는 사람이 요청을 하게 될때 /login 리디렉션을 시킨다는 뜻이고 그때 페이지는 /oauth2/authorization/keycloak
이 되는 것입니다 

정말 많이 한거 같지만 아직 init 절반도 못왔습니다 


```
OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest> accessTokenResponseClient = this.tokenEndpointConfig.accessTokenResponseClient;
if (accessTokenResponseClient == null) {
			accessTokenResponseClient = new DefaultAuthorizationCodeTokenResponseClient();
}

```

OAuth2AccessTokenResponseClient 이 부분은 차후에 승인코드를 받은 시큐리티가 accessToken 을 응답 받을떄 사용할 객체를 미리 정의해둡니다 
하단에 객체를 주는 DefaultAuthorizationCodeTokenResponseClient() 를 잠깐 살펴보면 

## DefaultAuthorizationCodeTokenResponseClient

```
public final class DefaultAuthorizationCodeTokenResponseClient{


    public DefaultAuthorizationCodeTokenResponseClient() {
		RestTemplate restTemplate = new RestTemplate(
				Arrays.asList(new FormHttpMessageConverter(), new OAuth2AccessTokenResponseHttpMessageConverter()));
		restTemplate.setErrorHandler(new OAuth2ErrorResponseErrorHandler());
		this.restOperations = restTemplate;
	}

	@Override
	public OAuth2AccessTokenResponse getTokenResponse(
			OAuth2AuthorizationCodeGrantRequest authorizationCodeGrantRequest) {
		Assert.notNull(authorizationCodeGrantRequest, "authorizationCodeGrantRequest cannot be null");
		RequestEntity<?> request = this.requestEntityConverter.convert(authorizationCodeGrantRequest);
		ResponseEntity<OAuth2AccessTokenResponse> response = getResponse(request);
		OAuth2AccessTokenResponse tokenResponse = response.getBody();
		if (CollectionUtils.isEmpty(tokenResponse.getAccessToken().getScopes())) {
			// As per spec, in Section 5.1 Successful Access Token Response
			// https://tools.ietf.org/html/rfc6749#section-5.1
			// If AccessTokenResponse.scope is empty, then default to the scope
			// originally requested by the client in the Token Request
			// @formatter:off
			tokenResponse = OAuth2AccessTokenResponse.withResponse(tokenResponse)
					.scopes(authorizationCodeGrantRequest.getClientRegistration().getScopes())
					.build();
			// @formatter:on
		}
		return tokenResponse;
	}

}


```

객체를 만들때에는 restTemplate 객체를 만들고 accesstoken 이 들어왔을때 어떻게 해석을 해석 토큰을 뽑아낼지를 재정의한 getTokenResponse 가 존재합니다 이 부분은 뒤에서도 나올 예정입니다 

다시 돌와와서 

```

OAuth2UserService<OAuth2UserRequest, OAuth2User> oauth2UserService = getOAuth2UserService();

```
oauth2UserService 는 accessToken 을 발급을 받고 이 토큰으로 사용자 정보를 요청할때 사용할 객체를 만들어두는 곳입니다 

마찬가지로 하단으로 내려가보면 

```

private OAuth2UserService<OAuth2UserRequest, OAuth2User> getOAuth2UserService() {

    if (this.userInfoEndpointConfig.userService != null) {
        return this.userInfoEndpointConfig.userService;
    }
    ResolvableType type = ResolvableType.forClassWithGenerics(OAuth2UserService.class, OAuth2UserRequest.class,
            OAuth2User.class);
    OAuth2UserService<OAuth2UserRequest, OAuth2User> bean = getBeanOrNull(type);
    if (bean != null) {
        return bean;
    }
    if (this.userInfoEndpointConfig.customUserTypes.isEmpty()) {
        return new DefaultOAuth2UserService();
    }
    List<OAuth2UserService<OAuth2UserRequest, OAuth2User>> userServices = new ArrayList<>();
    userServices.add(new CustomUserTypesOAuth2UserService(this.userInfoEndpointConfig.customUserTypes));
    userServices.add(new DefaultOAuth2UserService());
    return new DelegatingOAuth2UserService<>(userServices);
}

```

이렇게 기본적인 return new DefaultOAuth2UserService(); 로 객체를 만드는것을 확인할 수 있습니다 이 객체또한 잠깐 살펴보면 


## DefaultOAuth2UserService
```

public class DefaultOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {


    private Converter<OAuth2UserRequest, RequestEntity<?>> requestEntityConverter = new OAuth2UserRequestEntityConverter();

    private RestOperations restOperations;

    public DefaultOAuth2UserService() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setErrorHandler(new OAuth2ErrorResponseErrorHandler());
        this.restOperations = restTemplate;
    }


    @Override
	public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
		Assert.notNull(userRequest, "userRequest cannot be null");
		if (!StringUtils
				.hasText(userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint().getUri())) {
			OAuth2Error oauth2Error = new OAuth2Error(MISSING_USER_INFO_URI_ERROR_CODE,
					"Missing required UserInfo Uri in UserInfoEndpoint for Client Registration: "
							+ userRequest.getClientRegistration().getRegistrationId(),
					null);
			throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
		}
		String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint()
				.getUserNameAttributeName();
		if (!StringUtils.hasText(userNameAttributeName)) {
			OAuth2Error oauth2Error = new OAuth2Error(MISSING_USER_NAME_ATTRIBUTE_ERROR_CODE,
					"Missing required \"user name\" attribute name in UserInfoEndpoint for Client Registration: "
							+ userRequest.getClientRegistration().getRegistrationId(),
					null);
			throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
		}
		RequestEntity<?> request = this.requestEntityConverter.convert(userRequest);
		ResponseEntity<Map<String, Object>> response = getResponse(userRequest, request);
		Map<String, Object> userAttributes = response.getBody();
		Set<GrantedAuthority> authorities = new LinkedHashSet<>();
		authorities.add(new OAuth2UserAuthority(userAttributes));
		OAuth2AccessToken token = userRequest.getAccessToken();
		for (String authority : token.getScopes()) {
			authorities.add(new SimpleGrantedAuthority("SCOPE_" + authority));
		}
		return new DefaultOAuth2User(authorities, userAttributes, userNameAttributeName);
	}

}


```

객체를 만들때는 마찬가지로 RestTemplate 를 사용해서 만들고 하단에 보면 OAuth2User loadUser 가 정의되어 있는데 이 부분은 뒤에 자세하게 기술할 예정입니다 
access_token 을 통해서 user 정보를 가져온 시큐리티가 어떻게 유저정보를 파싱해서 시큐리티 안으로 ㅈ비어넣는지에 대한 코드가 적혀 있습니다 이 부분은 앞에서 시큐리티 form 로그인에서 본loadByUsername 하고 비슷한 로직을 가지게 됩니다 뒤에서 한번더 자세하게 다룰 예정입니다 


다시 돌아와서 

`OAuth2LoginAuthenticationProvider oauth2LoginAuthenticationProvider = new OAuth2LoginAuthenticationProvider(accessTokenResponseClient, oauth2UserService);`

그렇게 만들어진 accessTokenResponseClient oauth2UserService OAuth2LoginAuthenticationProvider 객체에 파라미터로 쓰이게 됩니다 OAuth2LoginAuthenticationProvider
는 다음시간에 나올 예정이니 자세하게 다룰 예정입니다 지금은 넘어가겠습니다 

```

GrantedAuthoritiesMapper userAuthoritiesMapper = this.getGrantedAuthoritiesMapper();
if (userAuthoritiesMapper != null) {
    oauth2LoginAuthenticationProvider.setAuthoritiesMapper(userAuthoritiesMapper);
}

```
그리고 form 로그인에서 보았던 권한 부여 mapper 은 여기도 쓰입니다 그런데 여기서는 일단 값이 null 인채로 넘어가게 됩니다 

`http.authenticationProvider(this.postProcess(oauth2LoginAuthenticationProvider));` 이 부분까지 와서는 이제 기본적인 로그인 절차 및 데이터 들어올 시 어떻게 파싱해서 인가서버와 주고받을지에 대한 모든 정보를 oauth2LoginAuthenticationProvider 담았음으로 이 부분 또한 authenticationProvider 에 담아주게 됩니다 

```

boolean oidcAuthenticationProviderEnabled = ClassUtils.isPresent("org.springframework.security.oauth2.jwt.JwtDecoder", this.getClass().getClassLoader());
if (oidcAuthenticationProviderEnabled) {
    OAuth2UserService<OidcUserRequest, OidcUser> oidcUserService = getOidcUserService();
    OidcAuthorizationCodeAuthenticationProvider oidcAuthorizationCodeAuthenticationProvider = new OidcAuthorizationCodeAuthenticationProvider(
            accessTokenResponseClient, oidcUserService);
    JwtDecoderFactory<ClientRegistration> jwtDecoderFactory = this.getJwtDecoderFactoryBean();
    if (jwtDecoderFactory != null) {
        oidcAuthorizationCodeAuthenticationProvider.setJwtDecoderFactory(jwtDecoderFactory);
    }
    if (userAuthoritiesMapper != null) {
        oidcAuthorizationCodeAuthenticationProvider.setAuthoritiesMapper(userAuthoritiesMapper);
    }
    http.authenticationProvider(this.postProcess(oidcAuthorizationCodeAuthenticationProvider));
}

``` 
이 부분은 인가서버는 로그인 방식이 2가지가 있는데 하나는 지금 할려고 하는 Oauth2 방식이 있고 다른 방식은 oidc 방식이 있습니다 그런데 oidc 방식은 
ClassUtils 에 org.springframework.security.oauth2.jwt.JwtDecoder 가 런타임으로 잡혀 있는지만 확인해서 없으면 이 방식으로는 진행을 하지 않습니다 
뒤에 이 방식을 활성화 시켜서 하는 것도 해볼예정입니다 


`this.initDefaultLoginFilter(http);` 그렇게 해서 이 모든 정보를 세팅을 하고 init 은 끝이나게 됩니다 

자 정말 길고 길었습니다 정리를 하자면 OAuth2LoginConfigurer 클래스는 정말 Oauth2 로그인 하기 전에 ClientRegistration 에 등록된 내용을 바탕으로 
로그인 페이지 , 로그인 방식 , 각 절차에 맞는 객체 생성 을 기본적으로 이곳에서 다 객체를 생성해서 올려놓고 진행을 하게 됩니다 

다음시간에는 정말로 로그인을 했을때 어떤일이 벌어지는지에 대해서 알아보도록 하겠습니다 








 


