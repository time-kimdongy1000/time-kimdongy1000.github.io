---
title: Spring Secuirty 25 OAuth2 OAuth2 -  OAuth2LoginAuthenticationFilter
author: kimdongy1000
date: 2023-07-03 14:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

우리는 지난시간에 로그인 url 이 어떻게 만들어지게 되었는지 OAuth2AuthorizationRequestRedirectFilter 를 통해서 알아보았습니다 


```
http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak

```

이 포스트는 실제 로그인을 하게 되어서 keyClock 가 전달해준 승인코드로 access_token 을 요청하는 로직을 따라가보겠습니다 
이는 OAuth2LoginConfigurer init 에 정의되어 있었던 OAuth2LoginAuthenticationFilter 쫒아가면 됩니다 
현재상황은 keyClock 이 제공하는 로그인 form 올 통해서 로그인 인증이 끝이나고 승인코드를 발급받은 상황입니다 

## OAuth2LoginAuthenticationFilter
```
@Override
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
		throws AuthenticationException {
	MultiValueMap<String, String> params = OAuth2AuthorizationResponseUtils.toMultiMap(request.getParameterMap());
	if (!OAuth2AuthorizationResponseUtils.isAuthorizationResponse(params)) {
		OAuth2Error oauth2Error = new OAuth2Error(OAuth2ErrorCodes.INVALID_REQUEST);
		throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
	}
	OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestRepository
			.removeAuthorizationRequest(request, response);
	if (authorizationRequest == null) {
		OAuth2Error oauth2Error = new OAuth2Error(AUTHORIZATION_REQUEST_NOT_FOUND_ERROR_CODE);
		throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
	}
	String registrationId = authorizationRequest.getAttribute(OAuth2ParameterNames.REGISTRATION_ID);
	ClientRegistration clientRegistration = this.clientRegistrationRepository.findByRegistrationId(registrationId);
	if (clientRegistration == null) {
		OAuth2Error oauth2Error = new OAuth2Error(CLIENT_REGISTRATION_NOT_FOUND_ERROR_CODE,
				"Client Registration not found with Id: " + registrationId, null);
		throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
	}
	// @formatter:off
	String redirectUri = UriComponentsBuilder.fromHttpUrl(UrlUtils.buildFullRequestUrl(request))
			.replaceQuery(null)
			.build()
			.toUriString();
	// @formatter:on
	OAuth2AuthorizationResponse authorizationResponse = OAuth2AuthorizationResponseUtils.convert(params,
			redirectUri);
	Object authenticationDetails = this.authenticationDetailsSource.buildDetails(request);
	OAuth2LoginAuthenticationToken authenticationRequest = new OAuth2LoginAuthenticationToken(clientRegistration,
			new OAuth2AuthorizationExchange(authorizationRequest, authorizationResponse));
	authenticationRequest.setDetails(authenticationDetails);
	OAuth2LoginAuthenticationToken authenticationResult = (OAuth2LoginAuthenticationToken) this
			.getAuthenticationManager().authenticate(authenticationRequest);
	OAuth2AuthenticationToken oauth2Authentication = this.authenticationResultConverter
			.convert(authenticationResult);
	Assert.notNull(oauth2Authentication, "authentication result cannot be null");
	oauth2Authentication.setDetails(authenticationDetails);
	OAuth2AuthorizedClient authorizedClient = new OAuth2AuthorizedClient(
			authenticationResult.getClientRegistration(), oauth2Authentication.getName(),
			authenticationResult.getAccessToken(), authenticationResult.getRefreshToken());

	this.authorizedClientRepository.saveAuthorizedClient(authorizedClient, oauth2Authentication, request, response);
	return oauth2Authentication;
}

```


```
MultiValueMap<String, String> params = OAuth2AuthorizationResponseUtils.toMultiMap(request.getParameterMap());
```

```

result = {ParameterMap}  size = 3
 "state" -> {String[1]} ["EExVByr9jatlV2D..."]
  key = "state"
  value = {String[1]} ["EExVByr9jatlV2D..."]
   0 = "EExVByr9jatlV2DMtNzqgYBUbA_bB_wpd1hykDryULc="
 "session_state" -> {String[1]} ["9c23d20c-d829-4..."]
  key = "session_state"
  value = {String[1]} ["9c23d20c-d829-4..."]
 "code" -> {String[1]} ["23048214-8b16-4..."]
  key = "code"
   value = {byte[4]} [99, 111, 100, 101]
   coder = 0
   hash = 3059181
  value = {String[1]} ["23048214-8b16-4..."]
   0 = "23048214-8b16-4eca-928e-a213753eedd9.9c23d20c-d829-4e8f-96af-38c934d758b0.4f8d60ae-7370-4bef-a356-95620f098f06"
```
현재 이 reslut 는 state 와 code 값이 들어 와 있습니다 즉 로그인은 성공을 했으니 access_token 인증 정보를 만들어서 요청을 하는것을 아래에서 진행을 합니다 


```

authorizationRequest = {OAuth2AuthorizationRequest} 
 authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
 authorizationGrantType = {AuthorizationGrantType} 
 responseType = {OAuth2AuthorizationResponseType} 
 clientId = "Spring-Oauth2-Authorizaion-client"
 redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
 scopes = {Collections$UnmodifiableSet}  size = 2
 state = "EExVByr9jatlV2DMtNzqgYBUbA_bB_wpd1hykDryULc="
 additionalParameters = {Collections$UnmodifiableMap}  size = 0
 authorizationRequestUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=EExVByr9jatlV2DMtNzqgYBUbA_bB_wpd1hykDryULc%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak"
 attributes = {Collections$UnmodifiableMap}  size = 1

```

객체 값을 보면 알겠지만 우리가 다 알고 있는 값들이 들어가 있습니다 다만 아직 인가서버에서 받은 code 값은 들어가 있지 않습니다 


```

String registrationId = authorizationRequest.getAttribute(OAuth2ParameterNames.REGISTRATION_ID);
ClientRegistration clientRegistration = this.clientRegistrationRepository.findByRegistrationId(registrationId);

```
reuqest 값을 분리해서 현재 사용하고 있는 인가서버가 무엇인지 찾고 있는데 이는 우리가 잘 알고 있는 clientRegistrationRepository 를 사용하고 있는것을 확인할 수 있습니다 


```
String redirectUri = UriComponentsBuilder.fromHttpUrl(UrlUtils.buildFullRequestUrl(request))
				.replaceQuery(null)
				.build()
				.toUriString();
OAuth2AuthorizationResponse authorizationResponse = OAuth2AuthorizationResponseUtils.convert(params,redirectUri);
```
request 정보에서 redirect 정보를 뽑아내고 이를 통해서 인가서버 응답을 담당할 객체 authorizationResponse 를 만들게 됩니다 


```
authorizationResponse = {OAuth2AuthorizationResponse} 
 redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
 state = "EExVByr9jatlV2DMtNzqgYBUbA_bB_wpd1hykDryULc="
 code = "23048214-8b16-4eca-928e-a213753eedd9.9c23d20c-d829-4e8f-96af-38c934d758b0.4f8d60ae-7370-4bef-a356-95620f098f06"
 error = null

```
이 authorizationResponse 에 redirect 가 있고 state 정보와 인가서버에서 받은 code 가 들어가게 됩니다 


```
OAuth2LoginAuthenticationToken authenticationRequest = new OAuth2LoginAuthenticationToken(clientRegistration,
				new OAuth2AuthorizationExchange(authorizationRequest, authorizationResponse));
```

그리고 위에서 만들었던 정보 clientRegistration ,  authorizationRequest , authorizationResponse 를 통해서 하나의 OAuth2LoginAuthenticationToken 타입의 객체를 만들게 되는데 이 토큰의 타입이 우리가 얻을 결과 값입니다 즉 시큐리티 context 에 들어가게 되는 토큰입니다 

```
OAuth2LoginAuthenticationToken authenticationResult = (OAuth2LoginAuthenticationToken) this.getAuthenticationManager().authenticate(authenticationRequest);
```

## ProviderManager 

```
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		
		...

		int size = this.providers.size();
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}

			...
		
			try {
				result = provider.authenticate(authentication);
				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}

			...
			
		}
		if (result == null && this.parent != null) {

			...
		
			try {
				parentResult = this.parent.authenticate(authentication);
				result = parentResult;
			}

			...
		}
		if (result != null) {
			if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
				
				((CredentialsContainer) result).eraseCredentials();
			}
			
			if (parentResult == null) {
				this.eventPublisher.publishAuthenticationSuccess(result);
			}

			return result;
		}

		
		if (lastException == null) {
			lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",
					new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
		}
		
		if (parentException == null) {
			prepareException(lastException, authentication);
		}
		throw lastException;
	}

}

```

이쪽으로 호출이 오게 되는데  우리 앞에서 form 인증할때도 보았던것이다 모든 시큐티리 manger 는 이쪽으로 들어와서 어떤 권한부여 클래스에서 호출될지 결정이 됩니다 

```
for (AuthenticationProvider provider : getProviders()) {
	if (!provider.supports(toTest)) {
		continue;
	}
}
```

이곳에서 어떤 AuthenticationProvider 사용할지 결정이 나게 되는데 

```

provider = {OAuth2LoginAuthenticationProvider} 
 authorizationCodeAuthenticationProvider = {OAuth2AuthorizationCodeAuthenticationProvider} 
 userService = {DefaultOAuth2UserService} 
 authoritiesMapper = {OAuth2LoginAuthenticationProvider$lambda} 

```
OAuth2LoginAuthenticationProvider 게 사용이 되는것을 알고 있으면된다 
그리고 하단 `result = provider.authenticate(authentication);` provider. 참조를 거는데 이때 provider -> OAuth2LoginAuthenticationProvider 가 되는것입니다 


## OAuth2LoginAuthenticationProvider
```
public class OAuth2LoginAuthenticationProvider implements AuthenticationProvider {

	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		OAuth2LoginAuthenticationToken loginAuthenticationToken = (OAuth2LoginAuthenticationToken) authentication;
		

		if (loginAuthenticationToken.getAuthorizationExchange().getAuthorizationRequest().getScopes()
				.contains("openid")) {
			
			return null;
		}
		OAuth2AuthorizationCodeAuthenticationToken authorizationCodeAuthenticationToken;
		try {
			authorizationCodeAuthenticationToken = (OAuth2AuthorizationCodeAuthenticationToken) this.authorizationCodeAuthenticationProvider
					.authenticate(new OAuth2AuthorizationCodeAuthenticationToken(
							loginAuthenticationToken.getClientRegistration(),
							loginAuthenticationToken.getAuthorizationExchange()));
		}

		...
		
	
		OAuth2AccessToken accessToken = authorizationCodeAuthenticationToken.getAccessToken();
		Map<String, Object> additionalParameters = authorizationCodeAuthenticationToken.getAdditionalParameters();
		OAuth2User oauth2User = this.userService.loadUser(new OAuth2UserRequest(
				loginAuthenticationToken.getClientRegistration(), accessToken, additionalParameters));
		Collection<? extends GrantedAuthority> mappedAuthorities = this.authoritiesMapper
				.mapAuthorities(oauth2User.getAuthorities());
		OAuth2LoginAuthenticationToken authenticationResult = new OAuth2LoginAuthenticationToken(
				loginAuthenticationToken.getClientRegistration(), loginAuthenticationToken.getAuthorizationExchange(),
				oauth2User, mappedAuthorities, accessToken, authorizationCodeAuthenticationToken.getRefreshToken());
		authenticationResult.setDetails(loginAuthenticationToken.getDetails());
		return authenticationResult;
	}
}

```

이곳으로 넘어오게 됩니다 

```
if (loginAuthenticationToken.getAuthorizationExchange().getAuthorizationRequest().getScopes()
			.contains("openid")) {
		return null;
}
```

이때 여기서 scopes 를 하나 읽게 되는데 이때 scopes 에서 openid 포함이 되어 있으면 이곳에서 처리 하지 않고 Oidc 쪽으로 넘기게 되는데 그 부분은 뒷장에 다룰 일이 있을것이니 일단 넘어갈것입니다 


```
authorizationCodeAuthenticationToken = (OAuth2AuthorizationCodeAuthenticationToken) this.authorizationCodeAuthenticationProvider.authenticate(new OAuth2AuthorizationCodeAuthenticationToken(loginAuthenticationToken.getClientRegistration(),loginAuthenticationToken.getAuthorizationExchange()));
```

이 부분이 이제 이 클래스에서 제일 중요한 부분이 되는것인데 


## OAuth2AuthorizationCodeAuthenticationProvider
```
@Override
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
	OAuth2AuthorizationCodeAuthenticationToken authorizationCodeAuthentication = (OAuth2AuthorizationCodeAuthenticationToken) authentication;
	OAuth2AuthorizationResponse authorizationResponse = authorizationCodeAuthentication.getAuthorizationExchange()
			.getAuthorizationResponse();

	...
	
	OAuth2AuthorizationRequest authorizationRequest = authorizationCodeAuthentication.getAuthorizationExchange()
			.getAuthorizationRequest();

	...

	OAuth2AccessTokenResponse accessTokenResponse = this.accessTokenResponseClient.getTokenResponse(
			new OAuth2AuthorizationCodeGrantRequest(authorizationCodeAuthentication.getClientRegistration(),
					authorizationCodeAuthentication.getAuthorizationExchange()));
	OAuth2AuthorizationCodeAuthenticationToken authenticationResult = new OAuth2AuthorizationCodeAuthenticationToken(
			authorizationCodeAuthentication.getClientRegistration(),
			authorizationCodeAuthentication.getAuthorizationExchange(), accessTokenResponse.getAccessToken(),
			accessTokenResponse.getRefreshToken(), accessTokenResponse.getAdditionalParameters());
	authenticationResult.setDetails(authorizationCodeAuthentication.getDetails());
	return authenticationResult;
}


```

이 부분이 핵심이 되는데 

```
OAuth2AccessTokenResponse accessTokenResponse = this.accessTokenResponseClient.getTokenResponse(new OAuth2AuthorizationCodeGrantRequest(authorizationCodeAuthentication.getClientRegistration(),authorizationCodeAuthentication.getAuthorizationExchange()));

```

accessTokenResponseClient 에 getTokenReposen 에 OAuth2AuthorizationCodeGrantRequest 에 ClientRegistration 과 
`authorizationCodeAuthentication.getAuthorizationExchange()` 넘기게 되는데 이곳에는 

```

result = {OAuth2AuthorizationExchange} 
 authorizationRequest = {OAuth2AuthorizationRequest} 
  authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
  authorizationGrantType = {AuthorizationGrantType} 
  responseType = {OAuth2AuthorizationResponseType} 
  clientId = "Spring-Oauth2-Authorizaion-client"
  redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
  scopes = {Collections$UnmodifiableSet}  size = 2
  state = "KCLHA3RKqrgw2GQq78OBZCY_-OvewjSj2yIV-Wm9Xh4="
  additionalParameters = {Collections$UnmodifiableMap}  size = 0
  authorizationRequestUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=KCLHA3RKqrgw2GQq78OBZCY_-OvewjSj2yIV-Wm9Xh4%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak"
  attributes = {Collections$UnmodifiableMap}  size = 1
 authorizationResponse = {OAuth2AuthorizationResponse} 
  redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
  state = "KCLHA3RKqrgw2GQq78OBZCY_-OvewjSj2yIV-Wm9Xh4="
  code = "f0367fd2-9c85-45a5-81e0-845ca081d9ea.9c23d20c-d829-4e8f-96af-38c934d758b0.4f8d60ae-7370-4bef-a356-95620f098f06"
  error = null

```
이런정보가 들어가 있습니다 즉 이 정보를 가지고 accessTokenResponseClient 에 이제 통신을 해서 토큰을 받아오게 됩니다 


## DefaultAuthorizationCodeTokenResponseClient 
```
public final class DefaultAuthorizationCodeTokenResponseClient
		implements OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest> {

	@Override
	public OAuth2AccessTokenResponse getTokenResponse(
			OAuth2AuthorizationCodeGrantRequest authorizationCodeGrantRequest) {
		Assert.notNull(authorizationCodeGrantRequest, "authorizationCodeGrantRequest cannot be null");
		RequestEntity<?> request = this.requestEntityConverter.convert(authorizationCodeGrantRequest);
		ResponseEntity<OAuth2AccessTokenResponse> response = getResponse(request);
		OAuth2AccessTokenResponse tokenResponse = response.getBody();
		if (CollectionUtils.isEmpty(tokenResponse.getAccessToken().getScopes())) {
			tokenResponse = OAuth2AccessTokenResponse.withResponse(tokenResponse)
					.scopes(authorizationCodeGrantRequest.getClientRegistration().getScopes())
					.build();
		}
		return tokenResponse;
	}
}

```

```
Assert.notNull(authorizationCodeGrantRequest, "authorizationCodeGrantRequest cannot be null");
RequestEntity<?> request = this.requestEntityConverter.convert(authorizationCodeGrantRequest);
ResponseEntity<OAuth2AccessTokenResponse> response = getResponse(request)

```

null 체크를 하고 이제 RequestEntity 에 request 정보를 심게 되는데 이 정보를 살펴보면

```

request = {RequestEntity@8170} "<POST http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token,{grant_type=[authorization_code], code=[e268bae3-fa08-490f-a95f-394873708412.9c23d20c-d829-4e8f-96af-38c934d758b0.4f8d60ae-7370-4bef-a356-95620f098f06], redirect_uri=[http://localhost:8081/login/oauth2/code/keycloak], client_id=[Spring-Oauth2-Authorizaion-client], client_secret=[NIe2qftuPcclGWFiBFicEWoK5SfYs7ql]},[Accept:"application/json;charset=UTF-8", Content-Type:"application/x-www-form-urlencoded;charset=UTF-8"]>"
 method = {HttpMethod@8174} "POST"
 url = {URI} "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token"
 type = null
 headers = {ReadOnlyHttpHeaders}  size = 2
  "Accept" -> {ArrayList}  size = 1
   key = "Accept"
   value = {ArrayList}  size = 1
    0 = "application/json;charset=UTF-8"
  "Content-Type" -> {ArrayList}  size = 1
   key = "Content-Type"
   value = {ArrayList}  size = 1
    0 = "application/x-www-form-urlencoded;charset=UTF-8"
 body = {LinkedMultiValueMap}  size = 5
  "grant_type" -> {ArrayList}  size = 1
   key = "grant_type"
   value = {ArrayList}  size = 1
    0 = "authorization_code"
  "code" -> {ArrayList}  size = 1
   key = "code"
   value = {ArrayList}  size = 1
    0 = "e268bae3-fa08-490f-a95f-394873708412.9c23d20c-d829-4e8f-96af-38c934d758b0.4f8d60ae-7370-4bef-a356-95620f098f06"
  "redirect_uri" -> {ArrayList}  size = 1
   key = "redirect_uri"
   value = {ArrayList}  size = 1
    0 = "http://localhost:8081/login/oauth2/code/keycloak"
  "client_id" -> {ArrayList}  size = 1
   key = "client_id"
   value = {ArrayList}  size = 1
    0 = "Spring-Oauth2-Authorizaion-client"
  "client_secret" -> {ArrayList}  size = 1
   key = "client_secret"
   value = {ArrayList}  size = 1
    0 = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
```

access_token 을 가져오기 위한 모든 데이터가 담겨 있습니다 기본적으로 token 요청을 할 url 부터 method 그리고 body 엔 grant 타입 그리고 code 그리고 clientId , clientSecret 까지 전부 담겨 있는 모습을 볼 수 있습니다 


```
OAuth2AccessTokenResponse tokenResponse = response.getBody();
```

그래서 통신이 성공하면 OAuth2AccessTokenResponse 타입의 객체를 하나 받게 되는데 이 테이터를 보게 되면 

```

accessToken = {OAuth2AccessToken@8290} 
	...
 tokenType = {OAuth2AccessToken$TokenType@8295} 

refreshToken = {OAuth2RefreshToken@8291} 
	...
```

우리가 그렇게 찾던 access_token 과 refresh 토큰이 같이 넘어오는것을 확인할 수 있었습니다 그리고 return 을 받게 되는데 
OAuth2AuthorizationCodeAuthenticationProvider 으로 돌아오게 됩니다 

```
OAuth2AccessTokenResponse accessTokenResponse = this.accessTokenResponseClient.getTokenResponse(
				new OAuth2AuthorizationCodeGrantRequest(authorizationCodeAuthentication.getClientRegistration(),
						authorizationCodeAuthentication.getAuthorizationExchange()));
```

여기 까지가 요청정보를 조합해서 인가서버와 통신해서 최종적으로 access_token 을 받아오는 것이고 

```

OAuth2AuthorizationCodeAuthenticationToken authenticationResult = new OAuth2AuthorizationCodeAuthenticationToken(
				authorizationCodeAuthentication.getClientRegistration(),
				authorizationCodeAuthentication.getAuthorizationExchange(), accessTokenResponse.getAccessToken(),
				accessTokenResponse.getRefreshToken(), accessTokenResponse.getAdditionalParameters());

```

바로 그 아래에선 이 clientRegistration 에 access_token 정보와 , refresh_token 정보를 저장을 하게 됩니다 그래서 최종적으로 이 객체에 저장된 정보를 찾아보면

```

authenticationResult = {OAuth2AuthorizationCodeAuthenticationToken@8330} "OAuth2AuthorizationCodeAuthenticationToken [Principal=Spring-Oauth2-Authorizaion-client, Credentials=[PROTECTED], Authenticated=true, Details=null, Granted Authorities=[]]"
 
 clientRegistration = {ClientRegistration} "ClientRegistration{registrationId='keycloak', clientId='Spring-Oauth2-Authorizaion-client', clientSecret='NIe2qftuPcclGWFiBFicEWoK5SfYs7ql', clientAuthenticationMethod=org.springframework.security.oauth2.core.ClientAuthenticationMethod@86baaa5b, authorizationGrantType=org.springframework.security.oauth2.core.AuthorizationGrantType@5da5e9f3, redirectUri='http://localhost:8081/login/oauth2/code/keycloak', scopes=[email, profile], providerDetails=org.springframework.security.oauth2.client.registration.ClientRegistration$ProviderDetails@bc237e5, 
 
 clientName='Spring-Oauth2-Authorizaion-client'}"
  registrationId = "keycloak"
  clientId = "Spring-Oauth2-Authorizaion-client"
  clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
  clientAuthenticationMethod = {ClientAuthenticationMethod} 
  authorizationGrantType = {AuthorizationGrantType} 
  redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
  scopes = {Collections$UnmodifiableSet}  size = 2
  providerDetails = {ClientRegistration$ProviderDetails} 
  clientName = "Spring-Oauth2-Authorizaion-client"
 
 authorizationExchange = {OAuth2AuthorizationExchange} 
 
 accessToken = {OAuth2AccessToken} 
  tokenType = {OAuth2AccessToken$TokenType} 
  scopes = {Collections$UnmodifiableSet}  size = 2
  tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY4NDQzODMsImlhdCI6MTY5Njg0NDA4MywiYXV0aF90aW1lIjoxNjk2ODQyNDU0LCJqdGkiOiIxYmQxYWVjYy04NDc2LTQyYWItOGU1MC1jOGU4YWM1NzE5NzYiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjljMjNkMjBjLWQ4MjktNGU4Zi05NmFmLTM4YzkzNGQ3NThiMCIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsInNpZCI6IjljMjNkMjBjLWQ4MjktNGU4Zi05NmFmLTM4YzkzNGQ3NThiMCIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFt"
  issuedAt = {Instant} "2023-10-09T09:34:43.901196300Z"
  expiresAt = {Instant} "2023-10-09T09:39:43.901196300Z"
 
 refreshToken = {OAuth2RefreshToken} 
  tokenValue = "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJmZTVmZjkwZC1lODA2LTQyMTEtYjBjZS1iZDdjZDkwNDcxMzAifQ.eyJleHAiOjE2OTY4NDU4ODMsImlhdCI6MTY5Njg0NDA4MywianRpIjoiYjhjZTg4ZWMtNGE2Yy00YzM3LWE2ZWUtOTQ5ZDRjMTQ2YWYzIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9TcnBpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1Qcm9qZWN0IiwiYXVkIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9TcnBpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1Qcm9qZWN0Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiUmVmcmVzaCIsImF6cCI6IlNwcmluZy1PYXV0aDItQXV0aG9yaXphaW9uLWNsaWVudCIsInNlc3Npb25fc3RhdGUiOiI5YzIzZDIwYy1kODI5LTRlOGYtOTZhZi0zOGM5MzRkNzU4YjAiLCJzY29wZSI6ImVtYWlsIHByb2ZpbGUiLCJzaWQiOiI5YzIzZDIwYy1kODI5LTRlOGYtOTZhZi0zOGM5MzRkNzU4YjAifQ.Az6iBpVIedCk7k2t7nRuo1cwKObrI6LSKgHt-TKIn28"
  issuedAt = {Instant} "2023-10-09T09:34:43.901196300Z"
  expiresAt = null
 authorities = {Collections$UnmodifiableRandomAccessList}  size = 0
 details = null
 authenticated = true

```

각 토큰의 토큰값을 포함한 발행일 만료일이 각각 정의되어 있고 이 토큰을 발행한 clientRegistraion 이 어디인지 저장하게 됩니다 차후 토큰이 만료되면 clientRegistraion 으로 한번더 통신을 할것이기 떄문에 이를 OAuth2AuthorizationCodeAuthenticationToken 타입의 객체에 담아놓게 됩니다 


그리고 이를 다시 return 해주는 곳은 OAuth2LoginAuthenticationProvider 인데 

```
try {
	authorizationCodeAuthenticationToken = (OAuth2AuthorizationCodeAuthenticationToken) this.authorizationCodeAuthenticationProvider
			.authenticate(new OAuth2AuthorizationCodeAuthenticationToken(
					loginAuthenticationToken.getClientRegistration(),
					loginAuthenticationToken.getAuthorizationExchange()));
}
catch (OAuth2AuthorizationException ex) {
	OAuth2Error oauth2Error = ex.getError();
	throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString(), ex);
}

```

이곳에서 요청정보를 만들어서 인가서버와 통신후에 access_token , refresh_token 을 발급받은 결과가 authorizationCodeAuthenticationToken 담겨 있고 이를 통해서
마지막 작업인 user 정보를 가져오는데 이때 access_token 을 사용하게 됩니다 그에 대한것은 다음시간에 다루는것으로 하겠습니다 

정리를 하겠습니다 
## access_token 
Authorization code grant 방식에서 2번째로 발급되는 토큰의 종류인데 인가서버에서 로그인시 return 되는 승인코드를 통해서 발급받을 수 있으며 이 token 을 통해서 
차후 user 정보를 받아 올 수 있는 토큰입니다 

## refresh_token 
access_token 이 만료 되었을때 이 refresh_token 을 통해서 재인증 절차없이 refresh_token 유효성만 검증되면 다시 access_token 을 발급받습니다 


요청순서 오늘 정말 여러 클래스를 넘나들면서 access_token 을 받아왔습니다 

1. OAuth2LoginAuthenticationFilter 에서 시작을 했습니다 

2. ProviderManager 를 통해서 어떤 곳에서 처리할지 정했는데 OAuth2LoginAuthenticationProvider 에서 처리하는 것으로 설정이 되었고 

3. OAuth2AuthorizationCodeAuthenticationProvider 에서 token 을 받아왔고 

4. 실제로 통신을 한 class 는 DefaultAuthorizationCodeTokenResponseClient 부분에서 통신을 해서 access_token 과 refresh 토큰을 발급받게 되었습니다 

생각보다 길고 어려운 내용입니다 저도 여러번 디버깅을 돌리면서 많이 해멨는데 다시 한번 정리하고 주기적으로 보는것으로 
access_token 이 어떻게 발급이 되는지 확인하였습니다