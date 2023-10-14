---

title: Spring Secuirty 29 OIDC 인증순서
author: kimdongy1000
date: 2023-07-05 16:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

지난시간에는 OIDC 가 무엇인지에 대해서 설명을 했습니다 Oauth2 는 제한된 권한으로 서드파티 어플리케이션에 접근해서 정보를 가져오는 반면에 
OIDC는 인증을 위한 프레임워크인 반면에 OAuth2 는 인가의 목적이 큰 프레임워크입니다 

## OIDC 설정 

```

spring.security.oauth2.client.registration.keycloak.scope=openid

```

## maven 추가 

```

<!-- https://mvnrepository.com/artifact/org.springframework.security/spring-security-oauth2-jose -->
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-oauth2-jose</artifactId>
</dependency>

```

application.properties 에서 scope 에 openid 라고 적어주게 되면 이제 이 시큐리는 OIDC 계층과 통신을 시도하게 됩니다 단 애초에 OIDC 자체가 OAUth2 위에 있기 때문에 사용하는 클래스는 동일합니다 그리고 이 maven 이 왜 필요한지는 이따 소스를 따라가면 나오게 됩니다 

OAuth2LoginConfigurer 이 클래스에서 init , configure 를 세팅을 하게 되고 

OAuth2AuthorizationRequestRedirectFilter 이곳에서 로그인 페이지 url 을 만들게 됩니다 

다만 여기서 좀 다른것은 

## OIDC 주소
```

http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=openid&state=3zWLz6U01v5b7tGnf_bWPxcDoqIUIyCoVtwRJbCR0bc%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak&nonce=38XBll3TR_ABtdeK3tfSx17CzgOBNOAdDWJHpws8nXo


```

scope 가 scope=openid 게 달라였습니다 앞에 OAuth2 에서는 

## OAuth2 주소

```

http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=email%20profile&state=KvR7XkC1EIAIjMCYxLr4Ljs_gzTuprTn5_tHWMrljY4%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak

```

scope=email%20profile 가 이렇게 이어져 있는것을 확인할 수 있습니다 

그럼 로그인을 진행하게되면 OAuth2LoginAuthenticationFilter 로 넘어가게 되는것이고 우리는 이 부분에서 자세히 보면됩니다 

## OAuth2LoginAuthenticationFilter

```

public class OAuth2LoginAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

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
}

```

위에부분은 앞에와 마찬가지로 ClientRegistraion 을 찾아오고 redirect url 그리고 요청객체 , 응답객체 만드는것은 동일하나 


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

실제로는 이 부분부터 달라질 예정입니다 자 그러면 이 부분터 보게 되면 

## ProviderManager

```

@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		AuthenticationException parentException = null;
		Authentication result = null;
		Authentication parentResult = null;
		int currentPosition = 0;
		int size = this.providers.size();
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}
			if (logger.isTraceEnabled()) {
				logger.trace(LogMessage.format("Authenticating request with %s (%d/%d)",
						provider.getClass().getSimpleName(), ++currentPosition, size));
			}
			try {
				result = provider.authenticate(authentication);
				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException | InternalAuthenticationServiceException ex) {
				prepareException(ex, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw ex;
			}
			catch (AuthenticationException ex) {
				lastException = ex;
			}
		}
		if (result == null && this.parent != null) {
			// Allow the parent to try.
			try {
				parentResult = this.parent.authenticate(authentication);
				result = parentResult;
			}
			catch (ProviderNotFoundException ex) {
				// ignore as we will throw below if no other exception occurred prior to
				// calling parent and the parent
				// may throw ProviderNotFound even though a provider in the child already
				// handled the request
			}
			catch (AuthenticationException ex) {
				parentException = ex;
				lastException = ex;
			}
		}
		if (result != null) {
			if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials();
			}
			// If the parent AuthenticationManager was attempted and successful then it
			// will publish an AuthenticationSuccessEvent
			// This check prevents a duplicate AuthenticationSuccessEvent if the parent
			// AuthenticationManager already published it
			if (parentResult == null) {
				this.eventPublisher.publishAuthenticationSuccess(result);
			}

			return result;
		}

		// Parent was null, or didn't authenticate (or throw an exception).
		if (lastException == null) {
			lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",
					new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
		}
		// If the parent AuthenticationManager was attempted and failed then it will
		// publish an AbstractAuthenticationFailureEvent
		// This check prevents a duplicate AbstractAuthenticationFailureEvent if the
		// parent AuthenticationManager already published it
		if (parentException == null) {
			prepareException(lastException, authentication);
		}
		throw lastException;
	}

```

ProviderManager 클래스의 목적은 현재 넘어오는 Oauth2 타입에 따라서 각기 다른 Provider 에서 처리하기 때문에 이곳에서 분기를 하게 됩니다 
그래서 앞에서는 OAuth2LoginAuthenticationProvider 로 처리하게끔 만들어졌다면 이번에는     

```

providers = {ArrayList@7943}  size = 3
 0 = {AnonymousAuthenticationProvider@8746} 
  messages = {MessageSourceAccessor@8750} 
  key = "9e2551b8-cdf1-41fc-8c31-af2a9e10aaf3"
 1 = {OAuth2LoginAuthenticationProvider@8696} 
  authorizationCodeAuthenticationProvider = {OAuth2AuthorizationCodeAuthenticationProvider@8752} 
  userService = {DefaultOAuth2UserService@8753} 
  authoritiesMapper = {OAuth2LoginAuthenticationProvider$lambda@8754} 
 2 = {OidcAuthorizationCodeAuthenticationProvider@7969} 
  accessTokenResponseClient = {DefaultAuthorizationCodeTokenResponseClient@8441} 
  userService = {OidcUserService@8439} 
  jwtDecoderFactory = {OidcIdTokenDecoderFactory@8442} 
  authoritiesMapper = {OidcAuthorizationCodeAuthenticationProvider$lambda@8440} 

```

총 3개가 준비가 되어 있고 이중에서 OidcAuthorizationCodeAuthenticationProvider 에서 처리될 예정입니다 

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

일단 한번은 무조건 호출이 될 예정인데 여기서 중요한것은 

```

if (loginAuthenticationToken.getAuthorizationExchange().getAuthorizationRequest().getScopes()
      .contains("openid")) {
    // This is an OpenID Connect Authentication Request so return null
    // and let OidcAuthorizationCodeAuthenticationProvider handle it instead
    return null;
  }


```

즉 요청정보에 opendid를 포함하고 있으면 이곳에서 처리를 하지 않고 return 하게 되는것입니다 

## OidcAuthorizationCodeAuthenticationProvider

```

public class OidcAuthorizationCodeAuthenticationProvider implements AuthenticationProvider {

  	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		OAuth2LoginAuthenticationToken authorizationCodeAuthentication = (OAuth2LoginAuthenticationToken) authentication;
		// Section 3.1.2.1 Authentication Request -
		// https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest
		// scope
		// REQUIRED. OpenID Connect requests MUST contain the "openid" scope value.
		if (!authorizationCodeAuthentication.getAuthorizationExchange().getAuthorizationRequest().getScopes()
				.contains(OidcScopes.OPENID)) {
			// This is NOT an OpenID Connect Authentication Request so return null
			// and let OAuth2LoginAuthenticationProvider handle it instead
			return null;
		}
		OAuth2AuthorizationRequest authorizationRequest = authorizationCodeAuthentication.getAuthorizationExchange()
				.getAuthorizationRequest();
		OAuth2AuthorizationResponse authorizationResponse = authorizationCodeAuthentication.getAuthorizationExchange()
				.getAuthorizationResponse();
		if (authorizationResponse.statusError()) {
			throw new OAuth2AuthenticationException(authorizationResponse.getError(),
					authorizationResponse.getError().toString());
		}
		if (!authorizationResponse.getState().equals(authorizationRequest.getState())) {
			OAuth2Error oauth2Error = new OAuth2Error(INVALID_STATE_PARAMETER_ERROR_CODE);
			throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
		}
		OAuth2AccessTokenResponse accessTokenResponse = getResponse(authorizationCodeAuthentication);
		ClientRegistration clientRegistration = authorizationCodeAuthentication.getClientRegistration();
		Map<String, Object> additionalParameters = accessTokenResponse.getAdditionalParameters();
		if (!additionalParameters.containsKey(OidcParameterNames.ID_TOKEN)) {
			OAuth2Error invalidIdTokenError = new OAuth2Error(INVALID_ID_TOKEN_ERROR_CODE,
					"Missing (required) ID Token in Token Response for Client Registration: "
							+ clientRegistration.getRegistrationId(),
					null);
			throw new OAuth2AuthenticationException(invalidIdTokenError, invalidIdTokenError.toString());
		}
		OidcIdToken idToken = createOidcToken(clientRegistration, accessTokenResponse);
		validateNonce(authorizationRequest, idToken);
		OidcUser oidcUser = this.userService.loadUser(new OidcUserRequest(clientRegistration,
				accessTokenResponse.getAccessToken(), idToken, additionalParameters));
		Collection<? extends GrantedAuthority> mappedAuthorities = this.authoritiesMapper
				.mapAuthorities(oidcUser.getAuthorities());
		OAuth2LoginAuthenticationToken authenticationResult = new OAuth2LoginAuthenticationToken(
				authorizationCodeAuthentication.getClientRegistration(),
				authorizationCodeAuthentication.getAuthorizationExchange(), oidcUser, mappedAuthorities,
				accessTokenResponse.getAccessToken(), accessTokenResponse.getRefreshToken());
		authenticationResult.setDetails(authorizationCodeAuthentication.getDetails());
		return authenticationResult;
	}


}

```

그렇기에 3번째 OidcAuthorizationCodeAuthenticationProvider 인 이곳에서 처리될 예정입니다 

```

if (!authorizationCodeAuthentication.getAuthorizationExchange().getAuthorizationRequest().getScopes()
    .contains(OidcScopes.OPENID)) {
  // This is NOT an OpenID Connect Authentication Request so return null
  // and let OAuth2LoginAuthenticationProvider handle it instead
  return null;
}

```

이곳도 마찬가지로 scope 에 opendid 가 포함이 되어 있지 않다면 이곳에서 처리 못하게끔 되어 있지만 우리는 이곳을 넘어갈 예정입니다 

```

OAuth2AuthorizationRequest authorizationRequest = authorizationCodeAuthentication.getAuthorizationExchange()
				.getAuthorizationRequest();
OAuth2AuthorizationResponse authorizationResponse = authorizationCodeAuthentication.getAuthorizationExchange()
    .getAuthorizationResponse();

```

그리고 통신하기 위해 요청객체 응답객체를 각각 만들게 됩니다 

```

authorizationRequest = {OAuth2AuthorizationRequest@7793} 
 authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
 authorizationGrantType = {AuthorizationGrantType@7837} 
 responseType = {OAuth2AuthorizationResponseType@7838} 
 clientId = "Spring-Oauth2-Authorizaion-client"
 redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
 scopes = {Collections$UnmodifiableSet@7841}  size = 1
  0 = "openid"
 state = "5-1ELnPeEb15bKd5db7afsulqyYwU0EWKBLoxlKyAAo="
  value = {byte[44]@7851} [53, 45, 49, 69, 76, 110, 80, 101, 69, 98, 49, 53, 98, 75, 100, 53, 100, 98, 55, 97, 102, 115, 117, 108, 113, 121, 89, 119, 85, 48, 69, 87, 75, 66, 76, 111, 120, 108, 75, 121, 65, 65, 111, 61]
  coder = 0
  hash = -479601817
 additionalParameters = {Collections$UnmodifiableMap@7843}  size = 1
  "nonce" -> "jxAd72Ms8h4kFuP_cR7ZplbiV4MNfURkn47ECcWbFDw"
 authorizationRequestUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth?response_type=code&client_id=Spring-Oauth2-Authorizaion-client&scope=openid&state=5-1ELnPeEb15bKd5db7afsulqyYwU0EWKBLoxlKyAAo%3D&redirect_uri=http://localhost:8081/login/oauth2/code/keycloak&nonce=jxAd72Ms8h4kFuP_cR7ZplbiV4MNfURkn47ECcWbFDw"
 attributes = {Collections$UnmodifiableMap@7845}  size = 2
  "registration_id" -> "keycloak"
  "nonce" -> "K7vZ7WvixvbO5ExawzScWoTmzqV8XLHhatZAQQEETXUuLtQa-uihBL2DxZ4nLX1HcRpCURF1rBMjjuovwJ237RykT-n8zAT39GPH18RF7pGNtxoHWNQ1cYJsJ44drhdC"

```

```

authorizationResponse = {OAuth2AuthorizationResponse@7797} 
 redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
 state = "5-1ELnPeEb15bKd5db7afsulqyYwU0EWKBLoxlKyAAo="
 code = "34e46a46-0eca-4571-87f5-d9d95a80aeb5.82543964-bd21-47b4-bb02-f849253de542.4f8d60ae-7370-4bef-a356-95620f098f06"
 error = null

```
여기서 중요한점은 이것도 마찬가지로 code 를 들고 있는 상태이고 이 code 를 던져서 id_token 을 받아올 예정인데 앞에서 Oauth2 에서는 code 를 던지면 
access_token 이 넘어온다는 점하고는 다릅니다 이토큰의 차이점도 앞에서 다루어 봤구요 


```

OAuth2AccessTokenResponse accessTokenResponse = getResponse(authorizationCodeAuthentication);
ClientRegistration clientRegistration = authorizationCodeAuthentication.getClientRegistration();

```

이곳에서 서버와 통신을 해서 accessTokenResponse 를 추출하게 되는데 이를 잘보면

```

accessTokenResponse = {OAuth2AccessTokenResponse@7932} 
 accessToken = {OAuth2AccessToken@7935} 
  tokenType = {OAuth2AccessToken$TokenType@7938} 
  scopes = {Collections$UnmodifiableSet@7939}  size = 3
  tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTcyNTgzNTYsImlhdCI6MTY5NzI1ODA1NiwiYXV0aF90aW1lIjoxNjk3MjU3MDcxLCJqdGkiOiI3NDMyYjg3Mi0zZTFiLTRmZjctODYyMS1mNjY3NDFmM2Q4MjAiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiU3ByaW5nLU9hdXRoMi1BdXRob3JpemFpb24tY2xpZW50Iiwibm9uY2UiOiJpMFY2MFl0Z3A2djh3enprRnlTajd0WVlUTHlVT2FyQXZNd2lma2h5d3RjIiwic2Vzc2lvbl9zdGF0ZSI6IjgyNTQzOTY0LWJkMjEtNDdiNC1iYjAyLWY4NDkyNTNkZTU0MiIsImFjciI6IjAiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtc3JwaW5nLW9hdXRoMi1hdXRob3JpemFpb24tcHJvamVjdCJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIGVtYWlsIHByb2ZpbGUiLCJzaWQiOiI4MjU0"
  issuedAt = {Instant@7941} "2023-10-14T04:34:16.314819200Z"
  expiresAt = {Instant@7942} "2023-10-14T04:39:16.314819200Z"

 refreshToken = {OAuth2RefreshToken@7936} 
  tokenValue = "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJmZTVmZjkwZC1lODA2LTQyMTEtYjBjZS1iZDdjZDkwNDcxMzAifQ.eyJleHAiOjE2OTcyNTk4NTYsImlhdCI6MTY5NzI1ODA1NiwianRpIjoiYjhkZWUzZDAtZWZlMy00MWQ0LTk2MTUtZDM4ZWVlZGNlODlmIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9TcnBpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1Qcm9qZWN0IiwiYXVkIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL3JlYWxtcy9TcnBpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1Qcm9qZWN0Iiwic3ViIjoiMzAyODMyMjgtZmEzNi00ZDg4LTgyYzMtYzk0OTRkYzQ0YmZkIiwidHlwIjoiUmVmcmVzaCIsImF6cCI6IlNwcmluZy1PYXV0aDItQXV0aG9yaXphaW9uLWNsaWVudCIsIm5vbmNlIjoiaTBWNjBZdGdwNnY4d3p6a0Z5U2o3dFlZVEx5VU9hckF2TXdpZmtoeXd0YyIsInNlc3Npb25fc3RhdGUiOiI4MjU0Mzk2NC1iZDIxLTQ3YjQtYmIwMi1mODQ5MjUzZGU1NDIiLCJzY29wZSI6Im9wZW5pZCBlbWFpbCBwcm9maWxlIiwic2lkIjoiODI1NDM5NjQtYmQyMS00N2I0LWJiMDItZjg0OTI1M2RlNTQyIn0.GfjdoHEVKbe3C3psbh0P0FPBWe1f5mShjAF30F1yx7g"
  issuedAt = {Instant@7941} "2023-10-14T04:34:16.314819200Z"
  expiresAt = null
 
 additionalParameters = {Collections$UnmodifiableMap@7937}  size = 4
  "refresh_expires_in" -> {Integer@7957} 1800
  "id_token" -> "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTcyNTgzNTYsImlhdCI6MTY5NzI1ODA1NiwiYXV0aF90aW1lIjoxNjk3MjU3MDcxLCJqdGkiOiJiMjRlMTEwNy1jMzU5LTRjMmQtYTM5Ni0wNzVkNWFhYmM2NDQiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJTcHJpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1jbGllbnQiLCJzdWIiOiIzMDI4MzIyOC1mYTM2LTRkODgtODJjMy1jOTQ5NGRjNDRiZmQiLCJ0eXAiOiJJRCIsImF6cCI6IlNwcmluZy1PYXV0aDItQXV0aG9yaXphaW9uLWNsaWVudCIsIm5vbmNlIjoiaTBWNjBZdGdwNnY4d3p6a0Z5U2o3dFlZVEx5VU9hckF2TXdpZmtoeXd0YyIsInNlc3Npb25fc3RhdGUiOiI4MjU0Mzk2NC1iZDIxLTQ3YjQtYmIwMi1mODQ5MjUzZGU1NDIiLCJhdF9oYXNoIjoiU3hQaU4xb2JEWmtPcHcwWW4wdGpEUSIsImFjciI6IjAiLCJzaWQiOiI4MjU0Mzk2NC1iZDIxLTQ3YjQtYmIwMi1mODQ5MjUzZGU1NDIiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsIm5hbWUiOiJ0aW1lIHVzZXIiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJ1c2VyMSIsImdpdmVuX25hbWUiOiJ0aW1lIiwiZmFtaWx5X25hbWUiOiJ1c2VyIiwiZW1haWwiOiJ1c2VyMUBnbWFpbC5jb20ifQ.D70n7bmPyaDTOwXiq"
  "not-before-policy" -> {Integer@7961} 0
  "session_state" -> "82543964-bd21-47b4-bb02-f849253de542"

```

여기서 잘보면 code 로 요청을 했기때문에 access_token 과 refresh_token 은 동일하게 오지만 추가적으로 이곳엔 id_token 이 하나가 발급이 됩니다 이 id 토큰을 jwt.io 에 분석해보면

```

{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "dp7lpFYPY-do8i9U6Vp3sqb4atyutsw1DUQtZZiwI_s"
}

{
  "exp": 1697258356,
  "iat": 1697258056,
  "auth_time": 1697257071,
  "jti": "b24e1107-c359-4c2d-a396-075d5aabc644",
  "iss": "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project",
  "aud": "Spring-Oauth2-Authorizaion-client",
  "sub": "30283228-fa36-4d88-82c3-c9494dc44bfd",
  "typ": "ID",
  "azp": "Spring-Oauth2-Authorizaion-client",
  "nonce": "i0V60Ytgp6v8wzzkFySj7tYYTLyUOarAvMwifkhywtc",
  "session_state": "82543964-bd21-47b4-bb02-f849253de542",
  "at_hash": "SxPiN1obDZkOpw0Yn0tjDQ",
  "acr": "0",
  "sid": "82543964-bd21-47b4-bb02-f849253de542",
  "email_verified": false,
  "name": "time user",
  "preferred_username": "user1",
  "given_name": "time",
  "family_name": "user",
  "email": "user1@gmail.com"
}


```

인데 여기서 중요한점은 실제로는 우리는 앞에서 access_token 을 보내야 위와 같은 데이터를 받을 수 있는 반면에 현재 oidc 에서는 code 를 보냄과 동시에 
인증이 완료되었다고 판단해서 keyclock 에서는 사용자 정보를 같이 access_token 과 refresh_token 를  반환을 해버립니다 이게 
앞에서 하는 OAuth2 와 OIDC 의 가장큰 차이점 입니다 


```

Map<String, Object> additionalParameters = accessTokenResponse.getAdditionalParameters();
  if (!additionalParameters.containsKey(OidcParameterNames.ID_TOKEN)) {
    OAuth2Error invalidIdTokenError = new OAuth2Error(INVALID_ID_TOKEN_ERROR_CODE,
        "Missing (required) ID Token in Token Response for Client Registration: "
            + clientRegistration.getRegistrationId(),
        null);

```

이제는 access_token 이 중요한것이 아니라  getAdditionalParameters 에서 ID_TOKEN 을 뽑아내는것이 더 중요하기 때문에 그것이 없으면 에러 처리 하게끔 설계가 되어 있습니다 

```

OidcIdToken idToken = createOidcToken(clientRegistration, accessTokenResponse);

```

이 정보를 활용해서 oidc 토큰을 만들게 되고 이때 createOidcToken 살펴보게 되면 


```

private OidcIdToken createOidcToken(ClientRegistration clientRegistration,
    OAuth2AccessTokenResponse accessTokenResponse) {
  JwtDecoder jwtDecoder = this.jwtDecoderFactory.createDecoder(clientRegistration);
  Jwt jwt = getJwt(accessTokenResponse, jwtDecoder);
  OidcIdToken idToken = new OidcIdToken(jwt.getTokenValue(), jwt.getIssuedAt(), jwt.getExpiresAt(),
      jwt.getClaims());
  return idToken;
}


```

JwtDecoder 객체를 활용하게 되는데 이때 우리가 앞에서 maven 으로 추가해준 라이브러리에 담겨 있는 것입니다 그래서 이 토큰을 jwtDecoder 돌려서

토큰의 value 와 , 토큰 발생시간 토큰 만료시간 토큰 클레임을 다 분리하고 그것 idToken 으로 만들어서 return 하게 됩니다 

```

OidcUser oidcUser = this.userService.loadUser(new OidcUserRequest(clientRegistration,accessTokenResponse.getAccessToken(), idToken, additionalParameters));

```

그리고 oidcUser 를 앞에서 Oauth2 User 만들듯이 loadUser 만들게 됩니다 

## OidcUserService 
```

public class OidcUserService implements OAuth2UserService<OidcUserRequest, OidcUser> {


  @Override
	public OidcUser loadUser(OidcUserRequest userRequest) throws OAuth2AuthenticationException {
		Assert.notNull(userRequest, "userRequest cannot be null");
		OidcUserInfo userInfo = null;
		if (this.shouldRetrieveUserInfo(userRequest)) {
			OAuth2User oauth2User = this.oauth2UserService.loadUser(userRequest);
			Map<String, Object> claims = getClaims(userRequest, oauth2User);
			userInfo = new OidcUserInfo(claims);
			// https://openid.net/specs/openid-connect-core-1_0.html#UserInfoResponse
			// 1) The sub (subject) Claim MUST always be returned in the UserInfo Response
			if (userInfo.getSubject() == null) {
				OAuth2Error oauth2Error = new OAuth2Error(INVALID_USER_INFO_RESPONSE_ERROR_CODE);
				throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
			}
			// 2) Due to the possibility of token substitution attacks (see Section
			// 16.11),
			// the UserInfo Response is not guaranteed to be about the End-User
			// identified by the sub (subject) element of the ID Token.
			// The sub Claim in the UserInfo Response MUST be verified to exactly match
			// the sub Claim in the ID Token; if they do not match,
			// the UserInfo Response values MUST NOT be used.
			if (!userInfo.getSubject().equals(userRequest.getIdToken().getSubject())) {
				OAuth2Error oauth2Error = new OAuth2Error(INVALID_USER_INFO_RESPONSE_ERROR_CODE);
				throw new OAuth2AuthenticationException(oauth2Error, oauth2Error.toString());
			}
		}
		Set<GrantedAuthority> authorities = new LinkedHashSet<>();
		authorities.add(new OidcUserAuthority(userRequest.getIdToken(), userInfo));
		OAuth2AccessToken token = userRequest.getAccessToken();
		for (String authority : token.getScopes()) {
			authorities.add(new SimpleGrantedAuthority("SCOPE_" + authority));
		}
		return getUser(userRequest, userInfo, authorities);
	}


}

```

```

OAuth2User oauth2User = this.oauth2UserService.loadUser(userRequest);
Map<String, Object> claims = getClaims(userRequest, oauth2User);
userInfo = new OidcUserInfo(claims);

```

이것은 몰랐는데 User를 만들떄에는 OAuth2User 기반으로 OAuth2USerservice 를 호출하게 됩니다 그리고 그것을 기반으로 claims 를 만들게 되는데 

```

userInfo = {OidcUserInfo@8509} 
 claims = {Collections$UnmodifiableMap@8511}  size = 7
  "sub" -> "30283228-fa36-4d88-82c3-c9494dc44bfd"
  "email_verified" -> {Boolean@8493} false
  "name" -> "time user"
  "preferred_username" -> "user1"
  "given_name" -> "time"
  "family_name" -> "user"
  "email" -> "user1@gmail.com"

```

유저 정보가 전부 이 claims 에 담기게 됩니다 

```

Set<GrantedAuthority> authorities = new LinkedHashSet<>();
authorities.add(new OidcUserAuthority(userRequest.getIdToken(), userInfo));
OAuth2AccessToken token = userRequest.getAccessToken();
for (String authority : token.getScopes()) {
  authorities.add(new SimpleGrantedAuthority("SCOPE_" + authority));
}
return getUser(userRequest, userInfo, authorities);

```

그리고 access_token 을 분리해서 권한을 만들어서 최종적으로 oidc 유저 타입의 객체를 만들게 됩니다 

```

OAuth2LoginAuthenticationToken authenticationResult = new OAuth2LoginAuthenticationToken(
				authorizationCodeAuthentication.getClientRegistration(),
				authorizationCodeAuthentication.getAuthorizationExchange(), oidcUser, mappedAuthorities,
				accessTokenResponse.getAccessToken(), accessTokenResponse.getRefreshToken());

```

그리고 되돌아와서는 이제까지 모인 정보로 authenticationResult 를 만들게 되고 

```

  claims = {Collections$UnmodifiableMap@8564}  size = 20
  tokenValue = "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJkcDdscEZZUFktZG84aTlVNlZwM3NxYjRhdHl1dHN3MURVUXRaWml3SV9zIn0.eyJleHAiOjE2OTcyNTg5NjAsImlhdCI6MTY5NzI1ODY2MCwiYXV0aF90aW1lIjoxNjk3MjU3MDcxLCJqdGkiOiI1NGUxZDUwOC1lYmYzLTQ3MTgtODBjNS1iYTY2NDJiMDVkY2IiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvcmVhbG1zL1NycGluZy1PYXV0aDItQXV0aG9yaXphaW9uLVByb2plY3QiLCJhdWQiOiJTcHJpbmctT2F1dGgyLUF1dGhvcml6YWlvbi1jbGllbnQiLCJzdWIiOiIzMDI4MzIyOC1mYTM2LTRkODgtODJjMy1jOTQ5NGRjNDRiZmQiLCJ0eXAiOiJJRCIsImF6cCI6IlNwcmluZy1PYXV0aDItQXV0aG9yaXphaW9uLWNsaWVudCIsIm5vbmNlIjoiNEZPVldSd1JPeHNYQk9QM01pM1hzc3FzbWduUDhHWWZZdUoxbU1vUWEtVSIsInNlc3Npb25fc3RhdGUiOiI4MjU0Mzk2NC1iZDIxLTQ3YjQtYmIwMi1mODQ5MjUzZGU1NDIiLCJhdF9oYXNoIjoiUWR2c1RPYTJGeGJZRURGc0Z4NUJfUSIsImFjciI6IjAiLCJzaWQiOiI4MjU0Mzk2NC1iZDIxLTQ3YjQtYmIwMi1mODQ5MjUzZGU1NDIiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsIm5hbWUiOiJ0aW1lIHVzZXIiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJ1c2VyMSIsImdpdmVuX25hbWUiOiJ0aW1lIiwiZmFtaWx5X25hbWUiOiJ1c2VyIiwiZW1haWwiOiJ1c2VyMUBnbWFpbC5jb20ifQ.EgkIyJ41IbV1n13ce"
  issuedAt = {Instant@8566} "2023-10-14T04:44:20Z"
  expiresAt = {Instant@8567} "2023-10-14T04:49:20Z"
 userInfo = {OidcUserInfo@8509} 
 authorities = {Collections$UnmodifiableSet@8535}  size = 4
 attributes = {Collections$UnmodifiableMap@8563}  size = 20
 nameAttributeKey = "sub"
clientRegistration = {ClientRegistration@7795} "ClientRegistration{registrationId='keycloak', clientId='Spring-Oauth2-Authorizaion-client', clientSecret='NIe2qftuPcclGWFiBFicEWoK5SfYs7ql', clientAuthenticationMethod=org.springframework.security.oauth2.core.ClientAuthenticationMethod@86baaa5b, authorizationGrantType=org.springframework.security.oauth2.core.AuthorizationGrantType@5da5e9f3, redirectUri='http://localhost:8081/login/oauth2/code/keycloak', scopes=[openid], providerDetails=org.springframework.security.oauth2.client.registration.ClientRegistration$ProviderDetails@1b2568f6, clientName='Spring-Oauth2-Authorizaion-client'}"
 registrationId = "keycloak"
 clientId = "Spring-Oauth2-Authorizaion-client"
 clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
 clientAuthenticationMethod = {ClientAuthenticationMethod@8581} 
 authorizationGrantType = {AuthorizationGrantType@8582} 
 redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
 scopes = {Collections$UnmodifiableSet@8584}  size = 1
 providerDetails = {ClientRegistration$ProviderDetails@8585} 
 clientName = "Spring-Oauth2-Authorizaion-client"
authorizationExchange = {OAuth2AuthorizationExchange@8550} 
authorities = {Collections$UnmodifiableRandomAccessList@8552}  size = 4
 0 = {OidcUserAuthority@8572} "ROLE_USER"
 1 = {SimpleGrantedAuthority@8573} "SCOPE_email"
 2 = {SimpleGrantedAuthority@8574} "SCOPE_openid"
 3 = {SimpleGrantedAuthority@8575} "SCOPE_profile"
details = null
authenticated = true

```

이렇게 복합적으로 담겨 있게 되고 

다시 OAuth2LoginAuthenticationFilter 로 돌아와서 

```
this.authorizedClientRepository.saveAuthorizedClient(authorizedClient, oauth2Authentication, request, response);

```

이제까지 모인 정보 들을 시큐리티 컨텍스트에 저장하면 끝이나게 됩니다 이렇게 되면 oidc 인증은 끝이나게 됩니다 

그리고 앞에서 만든 user 정보 가져오기를 비교해보면

```

Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_openid, SCOPE_profile]], User Attributes: [{at_hash=QdvsTOa2FxbYEDFsFx5B_Q, sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, iss=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project, typ=ID, preferred_username=user1, given_name=time, nonce=4FOVWRwROxsXBOP3Mi3XssqsmgnP8GYfYuJ1mMoQa-U, sid=82543964-bd21-47b4-bb02-f849253de542, aud=[Spring-Oauth2-Authorizaion-client], acr=0, azp=Spring-Oauth2-Authorizaion-client, auth_time=2023-10-14T04:17:51Z, name=time user, exp=2023-10-14T04:49:20Z, session_state=82543964-bd21-47b4-bb02-f849253de542, family_name=user, iat=2023-10-14T04:44:20Z, email=user1@gmail.com, jti=54e1d508-ebf3-4718-80c5-ba6642b05dcb}]


Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}]

```

첫번째것이 OIDC 정보이고 두번째가 OAuth2 를 활용해서 가져온 유저 정보이다 짝봐도 정보의 차이가 많이 나는것을 알 수 있다 
이때중요한것은 oidc 정보의 일부부인 oauth2 인부분도 있으니 둘을 완전히 다르다고는 할 수 없지만 사용하는 목적에 따라서 

oidc 를 쓸지 oauth2 를 쓸지가 결정이 되는것이다 

우리는 이렇게 oidc 로도 인증을 해서 유저 정보를 받아오는거 까지 해보았다 앞에 OAuth2 인증에서 많은 것을 보고 하나씩 다 까보고 진행을 했지만 oidc같은 경우는 거의 대부분을 Oauth2 로직과 비슷하게 가기 때문에 그 부분은 생략을 하고 oidc 관련한것들만 정리를 해보았습니다 











