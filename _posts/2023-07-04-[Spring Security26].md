---
title: Spring Secuirty 26 OAuth2 OAuth2 - OAuth2User
author: kimdongy1000
date: 2023-07-04 12:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

우리는 지난시간에 로그인을 성공해서 승인코드를 발급받고 그 코드를 이용해서 인가사버와 통신해서 access_token 과 refresh_token 을 발급을 받았습니다 
우리는 이 access_token 을 통해서 user 정보를 가져오는 코드를 같이 살펴볼 예정입니다 



## OAuth2LoginAuthenticationProvider
```
public class OAuth2LoginAuthenticationProvider implements AuthenticationProvider {


	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		OAuth2LoginAuthenticationToken loginAuthenticationToken = (OAuth2LoginAuthenticationToken) authentication;
		
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

우리는 지난시간까지 이 윗부분을 활용해서 통신결과 access_token 을 가져오는 것까지 살펴보았습니다 그 아래 부터 살펴보겠습니다 

```
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

```

```
OAuth2User oauth2User = this.userService.loadUser(new OAuth2UserRequest(loginAuthenticationToken.getClientRegistration(), accessToken, additionalParameters));
```

이때 이 loadUser 를 통과하게 되는데 우리가 앞에 시큐리티 form 로그인에서 loadByUsername 하고 동일한 역활을 하게 됩니다 

## DefaultOAuth2UserService

```
public class DefaultOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

	@Override
	public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
		
		...

		String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();
		
		...				
		
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
여기서 다시 인가서버와 통신할 준비를 하게 됩니다 이때는 userInfo 엔드포인트에 요청을 하게 됩니다 


```

RequestEntity<?> request = this.requestEntityConverter.convert(userRequest);

```

이때 RequestEntity 로 http 통신을 준비하게 됩니다 이때 userRequest 정보를 살펴보면 

```

userRequest = {OAuth2UserRequest} 
 clientName='{Spring-Oauth2-Authorizaion-client'}"
  registrationId = "keycloak"
  clientId = "Spring-Oauth2-Authorizaion-client"
  clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
  clientAuthenticationMethod = {ClientAuthenticationMethod} 
  authorizationGrantType = {AuthorizationGrantType} 
  redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
  scopes = {Collections$UnmodifiableSet}  size = 2
  providerDetails = {ClientRegistration$ProviderDetails} 
  clientName = "Spring-Oauth2-Authorizaion-client"
 
 accessToken = {OAuth2AccessToken} 
  tokenType = {OAuth2AccessToken$TokenType}  
  tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY5NDQ1MDQsImlhdCI6MTY5Njk0NDIwNCwiYXV0aF90aW1lIjoxNjk2OTQzOTQ5LCJqdGkiOiJkYTUxNzNjMy0wMjBhLTQ1ZGQtYmMyZi1iNjgxZGE3NTczNjciLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsInNpZCI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFt"
  issuedAt = {Instant@7986} "2023-10-10T13:23:24.453337500Z"
  expiresAt = {Instant@7987} "2023-10-10T13:28:24.453337500Z"
 

```
우리가 앞에서 인증이 완료된 ClientRegistraion 이 들어있고 access_token 으로 http 통신할 메세지를 만들게 됩니다 



그 결과는 이렇게 http 메세지가 만들어지게 됩니다 주소는 http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo
method 는 get 요청이 들어가게 되고 토큰은 헤더에 Bearer 방식으로 들어가게 됩니다 

```
ResponseEntity<Map<String, Object>> response = getResponse(userRequest, request);
Map<String, Object> userAttributes = response.getBody();
Set<GrantedAuthority> authorities = new LinkedHashSet<>();
```

그리고 요청을 보내게 되면 아래와 같이 getResponse 를 통해서 아래와 같은 결과가 나오게 되는데 


```
userAttributes = {LinkedHashMap}  size = 7
 "sub" -> "30283228-fa36-4d88-82c3-c9494dc44bfd"
 "email_verified" -> {Boolean} false
 "name" -> "time user"
 "preferred_username" -> "user1"
 "given_name" -> "time"
 "family_name" -> "user"
 "email" -> "user1@gmail.com"
```

boy 에서 이런 정보를 빼올 수 있습니다 이떄 sub 는 이 user 를 유일하게 식별할 수 있는 key 값이 되고 위에서 이 key 값으로 user 를 분리하게 됩니다 


```
Set<GrantedAuthority> authorities = new LinkedHashSet<>();
authorities.add(new OAuth2UserAuthority(userAttributes));
OAuth2AccessToken token = userRequest.getAccessToken();
for (String authority : token.getScopes()) {
	authorities.add(new SimpleGrantedAuthority("SCOPE_" + authority));
}
```

기본적으로는 `authorities.add(new OAuth2UserAuthority(userAttributes));` 통과시 ROLE_USER 가 만들어지는데 기 이유는 아래 OAuth2UserAuthority
에서 기본적으로 생성자 호출시 만들어지게 됩니다 

## OAuth2UserAuthority
```

public class OAuth2UserAuthority implements GrantedAuthority {

public OAuth2UserAuthority(Map<String, Object> attributes) {
		this("ROLE_USER", attributes);
	}
}

```

그리고 우리가 보낸 access_token 에서 scopes 를 분리해서 SCOPE_ 해서 권한을 더만들게 됩니다 우리는 앞에서 email , profile 를 했음으로 
이에 대한 권한이 들어가게 됩니다 

```
OAuth2AccessToken token = userRequest.getAccessToken();
for (String authority : token.getScopes()) {
	authorities.add(new SimpleGrantedAuthority("SCOPE_" + authority));
}

```

그래서 다시 return 을 받게 되는 이 부분에서는 

```

OAuth2User oauth2User = this.userService.loadUser(new OAuth2UserRequest(loginAuthenticationToken.getClientRegistration(), accessToken, additionalParameters));

```
아래와 같이 user 정보가 만들어지고 생겨나게 됩니다 

```

oauth2User = {DefaultOAuth2User@8094} "Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}]"
 authorities = {Collections$UnmodifiableSet@8100}  size = 3
  0 = {OAuth2UserAuthority@8060} "ROLE_USER"
  1 = {SimpleGrantedAuthority@8103} "SCOPE_email"
  2 = {SimpleGrantedAuthority@8104} "SCOPE_profile"
 attributes = {Collections$UnmodifiableMap@8101}  size = 7
  "sub" -> "30283228-fa36-4d88-82c3-c9494dc44bfd"
  "email_verified" -> {Boolean@8039} false
  "name" -> "time user"
  "preferred_username" -> "user1"
  "given_name" -> "time"
  "family_name" -> "user"
  "email" -> "user1@gmail.com"
 nameAttributeKey = "sub"
  value = {byte[3]@8116} [115, 117, 98]
  coder = 0
  hash = 114240

```

```

Collection<? extends GrantedAuthority> mappedAuthorities = this.authoritiesMapper
			.mapAuthorities(oauth2User.getAuthorities());
	OAuth2LoginAuthenticationToken authenticationResult = new OAuth2LoginAuthenticationToken(
			loginAuthenticationToken.getClientRegistration(), loginAuthenticationToken.getAuthorizationExchange(),
			oauth2User, mappedAuthorities, accessToken, authorizationCodeAuthenticationToken.getRefreshToken());
	authenticationResult.setDetails(loginAuthenticationToken.getDetails());

```


```
Collection<? extends GrantedAuthority> mappedAuthorities = this.authoritiesMapper.mapAuthorities(oauth2User.getAuthorities());

```

이 결과를 통해서 mappedAuthorities 로 권한을 포함하는 객체를 만들게 되고 그 하단 OAuth2LoginAuthenticationToken 객체는 이제까지 모두 모인 값 
ClientRegistrtaion , 요청때 사용한 정보 , 인증결과 user , 권한 그리고 access_token , refresh_token 을 포함한 거대한 객체 하나를 만들게 됩니다 


## OAuth2LoginAuthenticationFilter
```

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

```

이제 마지막으로 
```
OAuth2LoginAuthenticationToken authenticationResult = (OAuth2LoginAuthenticationToken) this
			.getAuthenticationManager().authenticate(authenticationRequest);

```

부분이 넘어오게 됩니다 위레서 만든 거대한 객체의 결과는 authenticationResult 에 담기게 됩니다 
그리고 시큐티티는 넘어오는 객체를 좀더 쉽게 사용하기 위해 아래의 메서드를 거치게 됩니다 

```
OAuth2AuthenticationToken oauth2Authentication = this.authenticationResultConverter.convert(authenticationResult);
```

그럼 객체는 아래와 같이 변하게 되며 이 결과를 최종적으로 시큐리티 context 에 담아주어야 합니다 

```

oauth2Authentication = {OAuth2AuthenticationToken@8500} "OAuth2AuthenticationToken [Principal=Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}], Credentials=[PROTECTED], Authenticated=true, Details=null, Granted Authorities=[ROLE_USER, SCOPE_email, SCOPE_profile]]"
 principal = {DefaultOAuth2User@8378} "Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}]"
 authorizedClientRegistrationId = "keycloak"
 authorities = {Collections$UnmodifiableRandomAccessList@8505}  size = 3
  0 = {OAuth2UserAuthority@8388} "ROLE_USER"
  1 = {SimpleGrantedAuthority@8389} "SCOPE_email"
  2 = {SimpleGrantedAuthority@8390} "SCOPE_profile"
 details = null
 authenticated = true

```

## AbstractAuthenticationProcessingFilter

```

public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean implements ApplicationEventPublisherAware, MessageSourceAware {

	protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
		Authentication authResult) throws IOException, ServletException {
		SecurityContext context = SecurityContextHolder.createEmptyContext();
		context.setAuthentication(authResult);
		SecurityContextHolder.setContext(context);
		this.securityContextRepository.saveContext(context, request, response);
		if (this.logger.isDebugEnabled()) {
			this.logger.debug(LogMessage.format("Set SecurityContextHolder to %s", authResult));
		}
		this.rememberMeServices.loginSuccess(request, response, authResult);
		if (this.eventPublisher != null) {
			this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(authResult, this.getClass()));
		}
		this.successHandler.onAuthenticationSuccess(request, response, authResult);
	}
		
}

```
인증이 성공되면 인증결과를 AbstractAuthenticationProcessingFilter 곳에서 인증컨텍스트에 심게 됩니다 

```

SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authResult);
SecurityContextHolder.setContext(context);

```

인증컨텍스트에 앞에서 넘어오는 결과를 심어두게 됩니다 


우리는 정말로 길고긴 소스를 보았습니다 그 첫번째인 Authorization_code_grant 방식에서 로그인 성공시 코드를 발급받고 , 그 코드를 통해서 access_token 을 발급받고 

이 access_token 을 통해서 userInfo 엔드포인트로 토큰을 전달해서 유저 정보를 전달하고 그 유저정보를 시큐리티 컨텍스트에 심는것으로 끝이나게 됩니다 

여기 까지 우리는 인가서버 KeyClock 를 활용해서 제 3의 서버에서 user 정보를 받아오는 것을 처음부터 끝까지 쭈욱 공부를 했습니다 다음장에서는 여기서 넘어오는 
user 를 우리가 어떻게 활용하는지에 대해서 알아보도록 하겠습니다 


