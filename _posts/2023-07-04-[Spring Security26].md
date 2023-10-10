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
		// Section 3.1.2.1 Authentication Request -
		// https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest scope
		// REQUIRED. OpenID Connect requests MUST contain the "openid" scope value.
		if (loginAuthenticationToken.getAuthorizationExchange().getAuthorizationRequest().getScopes()
				.contains("openid")) {
			// This is an OpenID Connect Authentication Request so return null
			// and let OidcAuthorizationCodeAuthenticationProvider handle it instead
			return null;
		}
		OAuth2AuthorizationCodeAuthenticationToken authorizationCodeAuthenticationToken;
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

우리는 지난시간까지 

```

OAuth2LoginAuthenticationToken loginAuthenticationToken = (OAuth2LoginAuthenticationToken) authentication;
// Section 3.1.2.1 Authentication Request -
// https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest scope
// REQUIRED. OpenID Connect requests MUST contain the "openid" scope value.
if (loginAuthenticationToken.getAuthorizationExchange().getAuthorizationRequest().getScopes()
		.contains("openid")) {
	// This is an OpenID Connect Authentication Request so return null
	// and let OidcAuthorizationCodeAuthenticationProvider handle it instead
	return null;
}
OAuth2AuthorizationCodeAuthenticationToken authorizationCodeAuthenticationToken;
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

이 윗부분을 활용해서 통신결과 access_token 을 가져오는 것까지 살펴보았습니다 그 아래 부터 살펴보겠습니다 


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

accessToken = {OAuth2AccessToken@7811} 
 tokenType = {OAuth2AccessToken$TokenType@7814} 
  value = "Bearer"
 scopes = {Collections$UnmodifiableSet@7815}  size = 2
  0 = "profile"
  1 = "email"
 tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY5NDQyNTksImlhdCI6MTY5Njk0Mzk1OSwiYXV0aF90aW1lIjoxNjk2OTQzOTQ5LCJqdGkiOiJjN2U4MWRkOC0xY2JlLTQxNjktOTk0Ny1iZjdjNWNhYmMzYjIiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImFjciI6IjEiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsInNpZCI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFt"
  value = {byte[1491]@7828} [101, 121, 74, 104, 98, 71, 99, 105, 79, 105, 74, 83, 85, 122, 73, 49, 78, 105, 73, 115, 73, 110, 82, 53, 99, 67, 73, 103, 79, 105, 65, 105, 83, 108, 100, 85, 73, 105, 119, 105, 97, 50, 108, 107, 73, 105, 65, 54, 73, 67, 74, 107, 99, 68, 100, 115, 99, 69, 90, 90, 85, 70, 107, 116, 90, 71, 56, 52, 97, 84, 108, 86, 78, 108, 90, 119, 77, 51, 78, 120, 89, 106, 82, 104, 100, 72, 108, 49, 100, 72, 78, 51, 77, 85, 82, 86, 85, 88, 82, 97, +1,391 more]
  coder = 0
  hash = 0
 issuedAt = {Instant@7817} "2023-10-10T13:19:19.650787600Z"
 expiresAt = {Instant@7818} "2023-10-10T13:24:19.650787600Z"

```

유저 정보를 가져올떄는 access_token 만 필요함으로 이에 대한 정보만 뽑아오게 됩니다 토큰은 Bearer 토큰으로 전달될것이고 이를 header 에 심어서 유저 정보를 받아옵니다 
그리고 tokenValue 가 실제 토큰의 현재값이고 scope 는 권한의 범위 그리고 발행일자인 issuedAt , 만료시간을 볼 수 있습니다 아무런 설정을 하지 않으면 access_token 은 5분 만료입니다 


```

OAuth2User oauth2User = this.userService.loadUser(new OAuth2UserRequest(loginAuthenticationToken.getClientRegistration(), accessToken, additionalParameters));

```
이때 이 loadUser 를 통과하게 되는데 우리가 앞에 시큐리티 form 로그인에서 loadByUsername 하고 동일한 역활을 하게 됩니다 

## DefaultOAuth2UserService

```

public class DefaultOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {


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

여기서 다시 인가서버와 통신할 준비를 하게 됩니다 

```

RequestEntity<?> request = this.requestEntityConverter.convert(userRequest);

```

이때 RequestEntity 로 http 통신을 준비하게 됩니다 이때 userRequest 정보를 살펴보면 

```

userRequest = {OAuth2UserRequest@7968} 
 clientRegistration = {ClientRegistration@7971} "ClientRegistration{registrationId='keycloak', clientId='Spring-Oauth2-Authorizaion-client', clientSecret='NIe2qftuPcclGWFiBFicEWoK5SfYs7ql', clientAuthenticationMethod=org.springframework.security.oauth2.core.ClientAuthenticationMethod@86baaa5b, authorizationGrantType=org.springframework.security.oauth2.core.AuthorizationGrantType@5da5e9f3, redirectUri='http://localhost:8081/login/oauth2/code/keycloak', scopes=[email, profile], providerDetails=org.springframework.security.oauth2.client.registration.ClientRegistration$ProviderDetails@102e007f, clientName='Spring-Oauth2-Authorizaion-client'}"
  registrationId = "keycloak"
  clientId = "Spring-Oauth2-Authorizaion-client"
  clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
  clientAuthenticationMethod = {ClientAuthenticationMethod@7978} 
  authorizationGrantType = {AuthorizationGrantType@7979} 
  redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
  scopes = {Collections$UnmodifiableSet@7981}  size = 2
  providerDetails = {ClientRegistration$ProviderDetails@7982} 
  clientName = "Spring-Oauth2-Authorizaion-client"
 
 accessToken = {OAuth2AccessToken@7972} 
  tokenType = {OAuth2AccessToken$TokenType@7814} 
  scopes = {Collections$UnmodifiableSet@7984}  size = 2
  tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY5NDQ1MDQsImlhdCI6MTY5Njk0NDIwNCwiYXV0aF90aW1lIjoxNjk2OTQzOTQ5LCJqdGkiOiJkYTUxNzNjMy0wMjBhLTQ1ZGQtYmMyZi1iNjgxZGE3NTczNjciLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsInNpZCI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFt"
  issuedAt = {Instant@7986} "2023-10-10T13:23:24.453337500Z"
  expiresAt = {Instant@7987} "2023-10-10T13:28:24.453337500Z"
 additionalParameters = {Collections$UnmodifiableMap@7973}  size = 3
  "session_state" -> "9324fde3-e580-4365-a24a-dccc288d1807"
  "refresh_expires_in" -> {Integer@7998} 1800
  "not-before-policy" -> {Integer@8000} 0

```


우리가 앞에서 인증이 완료된 ClientRegistraion 이 들어있고 access_token 으로 http 통신할 메세지를 만들게 됩니다 

```

request = {RequestEntity@8001} "<GET http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo,[Accept:"application/json", Authorization:"Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY5NDQ1MDQsImlhdCI6MTY5Njk0NDIwNCwiYXV0aF90aW1lIjoxNjk2OTQzOTQ5LCJqdGkiOiJkYTUxNzNjMy0wMjBhLTQ1ZGQtYmMyZi1iNjgxZGE3NTczNjciLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIs"
 method = {HttpMethod@8006} "GET"
 url = {URI@8007} "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo"
 type = null
 headers = {ReadOnlyHttpHeaders@8008}  size = 2
  "Accept" -> {ArrayList@8016}  size = 1
   key = "Accept"
   value = {ArrayList@8016}  size = 1
  "Authorization" -> {ArrayList@8018}  size = 1
   key = "Authorization"
   value = {ArrayList@8018}  size = 1
    0 = "Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY5NDQ1MDQsImlhdCI6MTY5Njk0NDIwNCwiYXV0aF90aW1lIjoxNjk2OTQzOTQ5LCJqdGkiOiJkYTUxNzNjMy0wMjBhLTQ1ZGQtYmMyZi1iNjgxZGE3NTczNjciLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsInNpZCI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZ"
 body = null

```

그 결과는 이렇게 http 메세지가 만들어지게 됩니다 주소는 http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo
method 는 get 요청이 들어가게 되고 토큰은 헤더에 Bearer 방식으로 들어가게 됩니다 

```

ResponseEntity<Map<String, Object>> response = getResponse(userRequest, request);
Map<String, Object> userAttributes = response.getBody();
Set<GrantedAuthority> authorities = new LinkedHashSet<>();

```
그리고 응답이 끝이나게 되면 

```

userAttributes = {LinkedHashMap@8025}  size = 7
 "sub" -> "30283228-fa36-4d88-82c3-c9494dc44bfd"
 "email_verified" -> {Boolean@8039} false
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

이 결괄르 통해서 mappedAuthorities 로 권한을 포함하는 객체를 만들게 되고 그 하단 OAuth2LoginAuthenticationToken 객체는 이제까지 모두 모인 값 
ClientRegistrtaion , 요청때 사용한 정보 , 인증결과 user , 권한 그리고 access_token , refresh_token 을 포함한 거대한 객체 하나를 만들게 됩니다 

```

authenticationResult = {OAuth2LoginAuthenticationToken@8397} "OAuth2LoginAuthenticationToken [Principal=Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted 
Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}], Credentials=[PROTECTED], Authenticated=true, Details=null, Granted 

Authorities=[ROLE_USER, SCOPE_email, SCOPE_profile]]"
 principal = {DefaultOAuth2User@8378} "Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}]"
  authorities = {Collections$UnmodifiableSet@8384}  size = 3
   0 = {OAuth2UserAuthority@8388} "ROLE_USER"
   1 = {SimpleGrantedAuthority@8389} "SCOPE_email"
   2 = {SimpleGrantedAuthority@8390} "SCOPE_profile"
  attributes = {Collections$UnmodifiableMap@8406}  size = 7
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
 
 clientRegistration = {ClientRegistration@8411} "ClientRegistration{registrationId='keycloak', clientId='Spring-Oauth2-Authorizaion-client', clientSecret='NIe2qftuPcclGWFiBFicEWoK5SfYs7ql', clientAuthenticationMethod=org.springframework.security.oauth2.core.ClientAuthenticationMethod@86baaa5b, authorizationGrantType=org.springframework.security.oauth2.core.AuthorizationGrantType@5da5e9f3, redirectUri='http://localhost:8081/login/oauth2/code/keycloak', scopes=[email, profile], providerDetails=org.springframework.security.oauth2.client.registration.ClientRegistration$ProviderDetails@102e007f, 
 
 clientName='Spring-Oauth2-Authorizaion-client'}"
  registrationId = "keycloak"
  clientId = "Spring-Oauth2-Authorizaion-client"
  clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
  clientAuthenticationMethod = {ClientAuthenticationMethod@8443} 
  authorizationGrantType = {AuthorizationGrantType@8444} 
  redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
  scopes = {Collections$UnmodifiableSet@8446}  size = 2
  providerDetails = {ClientRegistration$ProviderDetails@8447} 
  clientName = "Spring-Oauth2-Authorizaion-client"
 
 authorizationExchange = {OAuth2AuthorizationExchange@8400} 
  authorizationRequest = {OAuth2AuthorizationRequest@8401} 
   authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
   authorizationGrantType = {AuthorizationGrantType@8429} 
   responseType = {OAuth2AuthorizationResponseType@8430} 
   clientId = "Spring-Oauth2-Authorizaion-client"
   redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
   scopes = {Collections$UnmodifiableSet@8433}  size = 2
   state = "ScVvOSs5og581ufjoOR1rAWTvyxeGebadDY2aDcSso4="
   additionalParameters = {Collections$UnmodifiableMap@8435}  size = 0
   authorizationRequestUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=ScVvOSs5og581ufjoOR1rAWTvyxeGebadDY2aDcSso4%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak"
   attributes = {Collections$UnmodifiableMap@8437}  size = 1
  authorizationResponse = {OAuth2AuthorizationResponse@8402} 
   redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
   state = "ScVvOSs5og581ufjoOR1rAWTvyxeGebadDY2aDcSso4="
   code = "ea4549ab-365f-4925-b7e3-2e1e124f0566.9324fde3-e580-4365-a24a-dccc288d1807.4f8d60ae-7370-4bef-a356-95620f098f06"
   error = null
 
 accessToken = {OAuth2AccessToken@8333} 
  tokenType = {OAuth2AccessToken$TokenType@8416} 
  scopes = {Collections$UnmodifiableSet@8417}  size = 2
  tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTY5NDU0NTIsImlhdCI6MTY5Njk0NTE1MiwiYXV0aF90aW1lIjoxNjk2OTQzOTQ5LCJqdGkiOiI4N2U5ZjYxOS01NTgyLTQzODEtYTg2MS0zZWZlZDE4NzJhNTkiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsInNpZCI6IjkzMjRmZGUzLWU1ODAtNDM2NS1hMjRhLWRjY2MyODhkMTgwNyIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFt"
  issuedAt = {Instant@8419} "2023-10-10T13:39:12.334461200Z"
  expiresAt = {Instant@8420} "2023-10-10T13:44:12.334461200Z"
 refreshToken = {OAuth2RefreshToken@8412} 
  tokenValue = "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJmZTVmZjkwZC1lODA2LTQyMTEtYjBjZS1iZDdjZDkwNDcxMzAifQ.eyJleHAiOjE2OTY5NDY5NTIsImlhdCI6MTY5Njk0NTE1MiwianRpIjoiZTE2NmY2YmMtZDllNC00OWNiLWEzMDItM2EzOTg0YjNlZTgwIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9TcnBpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1Qcm9qZWN0IiwiYXVkIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9TcnBpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1Qcm9qZWN0Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiUmVmcmVzaCIsImF6cCI6IlNwcmluZy1PYXV0aDItQXV0aG9yaXphaW9uLWNsaWVudCIsInNlc3Npb25fc3RhdGUiOiI5MzI0ZmRlMy1lNTgwLTQzNjUtYTI0YS1kY2NjMjg4ZDE4MDciLCJzY29wZSI6ImVtYWlsIHByb2ZpbGUiLCJzaWQiOiI5MzI0ZmRlMy1lNTgwLTQzNjUtYTI0YS1kY2NjMjg4ZDE4MDcifQ.8fRH-SsO-ZUgKgIBEsz4ht9v-3OHAGDsdjU9erDcjcg"
  issuedAt = {Instant@8419} "2023-10-10T13:39:12.334461200Z"
  expiresAt = null
 
 authorities = {Collections$UnmodifiableRandomAccessList@8413}  size = 3
  0 = {OAuth2UserAuthority@8388} "ROLE_USER"
  1 = {SimpleGrantedAuthority@8389} "SCOPE_email"
  2 = {SimpleGrantedAuthority@8390} "SCOPE_profile"
 details = null
 authenticated = true

```

값이 좀 길어보이지만 key 값으로 하나씩 분리하면 우리가 이제까지 해왔던 모든 값들이 authenticationResult 결과로 만들어지게 됩니다 

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

이제까지 부분은 이 클래스에서 
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

```

authResult = {OAuth2AuthenticationToken@8500} "OAuth2AuthenticationToken [Principal=Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}], Credentials=[PROTECTED], Authenticated=true, Details=WebAuthenticationDetails [RemoteIpAddress=0:0:0:0:0:0:0:1, SessionId=D0786A396148EF49B36E9B7F1B81E1EA], Granted Authorities=[ROLE_USER, SCOPE_email, SCOPE_profile]]"
 principal = {DefaultOAuth2User@8378} "Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}]"
  authorities = {Collections$UnmodifiableSet@8384}  size = 3
  attributes = {Collections$UnmodifiableMap@8406}  size = 7
  nameAttributeKey = "sub"
 authorizedClientRegistrationId = "keycloak"
 authorities = {Collections$UnmodifiableRandomAccessList@8505}  size = 3
  0 = {OAuth2UserAuthority@8388} "ROLE_USER"
  1 = {SimpleGrantedAuthority@8389} "SCOPE_email"
  2 = {SimpleGrantedAuthority@8390} "SCOPE_profile"
 details = {WebAuthenticationDetails@8493} "WebAuthenticationDetails [RemoteIpAddress=0:0:0:0:0:0:0:1, SessionId=D0786A396148EF49B36E9B7F1B81E1EA]"
 authenticated = true

```

oauth2Authentication 의 데이터와 동일한 데이터를 시큐리티 컨텍스트에 심고 끝이나게 됩니다 


우리는 정말로 길고긴 소스를 보았습니다 그 첫번째인 Authorization_code_grant 방식에서 로그인 성공시 코드를 발급받고 , 그 코드를 통해서 access_token 을 발급받고 

이 access_token 을 통해서 userInfo 엔드포인트로 토큰을 전달해서 유저 정보를 전달하고 그 유저정보를 시큐리티 컨텍스트에 심는것으로 끝이나게 됩니다 

여기 까지 우리는 인가서버 KeyClock 를 활용해서 제 3의 서버에서 user 정보를 받아오는 것을 처음부터 끝까지 쭈욱 공부를 했습니다 다음장에서는 여기서 넘어오는 
user 를 우리가 어떻게 활용하는지에 대해서 알아보도록 하겠습니다 


