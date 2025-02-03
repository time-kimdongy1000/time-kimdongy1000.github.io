---
title: Spring Secuirty 33 Resource Server 검증절차
author: kimdongy1000
date: 2023-07-14 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

우리는 지난시간에 간단한 KeyClock 와 Resource - Server 연동과 권한이 없을때 리소스 서버가 행동하는 EntryPoint 에 대해서 알아보았습니다 이제는 우리가 
KeyClock 에서 받은 Resource - Server 에 토큰을 던저서 그 검증절차를 한번 다루어보겠습니다 

우리는 권한부여방식은 Authorization code grant 방식을 사용할 예정입니다 임시코드 및 access_token 발급은 
<https://time-kimdongy1000.github.io/posts/Spring-Security24/> 참고하시면됩니다 

검증절차는 다음과 같습니다 

1. 리소스 서버는 클라이언트로 부터 Access_token 을 받습니다 이때 Access_token 형태는 JWT 형태입니다
2. 리소스 서버는 이 토큰을 검증할때 인가서버로 부터 공개Key 를 발급을 받습니다 
3. 공개Key 로 JWT verify 가 true 로 return 되면 이 토큰은 유효한 토큰입니다 
4. JWT 를 각각 클레임셋으로 분리해서 시큐리티 컨텍스트에 넣고 마무리 하고 클라이언트가 최초로 요청한 곳으로 리다이렉트 하게 됩니다 



## BearerTokenAuthenticationFilter

```

@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
	throws ServletException, IOException {
		String token;
		try {
			token = this.bearerTokenResolver.resolve(request);
		}

		...

		context.setAuthentication(authenticationResult);
		SecurityContextHolder.setContext(context);
}

```

제일 먼저 토큰은 BearerTokenAuthenticationFilter 으로 모든 요청이 들어오게 됩니다 그리고 request 에 있는 토큰을 분리해서 하단에 있는 
`SecurityContextHolder.setContext(context);` 하게 되면 인증은 끝이나게 됩니다 

## DefaultBearerTokenResolver

```

@Override
public String resolve(final HttpServletRequest request) {
	final String authorizationHeaderToken = resolveFromAuthorizationHeader(request);
	
	...

	if (authorizationHeaderToken != null) {
		
		...

		return authorizationHeaderToken;
	}
	
	return null;
}

```
여기서는 간단하게 헤더에 있는 토큰을 분리하게 됩니다 토큰은 Authorization 가 key 값으로 있으며 있떄 시작은 반드시 bearer ~ 이렇게 시작을 하게 됩니다 그렇지 않으면 토큰 파싱에 실패해서 오류가 나게 됩니다 

## BearerTokenAuthenticationFilter

```

try {
	AuthenticationManager authenticationManager = this.authenticationManagerResolver.resolve(request);
	Authentication authenticationResult = authenticationManager.authenticate(authenticationRequest);
	SecurityContext context = SecurityContextHolder.createEmptyContext();
	context.setAuthentication(authenticationResult);
	SecurityContextHolder.setContext(context);
	this.securityContextRepository.saveContext(context, request, response);
	if (this.logger.isDebugEnabled()) {
		this.logger.debug(LogMessage.format("Set SecurityContextHolder to %s", authenticationResult));
	}
	filterChain.doFilter(request, response);
}

```
토큰은 return 받고 이곳으로 들어오게 되면 이곳에서 이제 모든을 것을 진행을 하게 됩니다 마찬가지로 인증매니저를 설정해서 어디에서 처리할것인지 정하게 되는데 
`this.authenticationManagerResolver.resolve(request);` 호출하게 되면 여러개의 Provider 중에 JwtAuthenticationProvider 를 선택을 하게 됩니다 

## JwtAuthenticationProvider

```

public Authentication authenticate(Authentication authentication) throws AuthenticationException {
	BearerTokenAuthenticationToken bearer = (BearerTokenAuthenticationToken) authentication;
	Jwt jwt = getJwt(bearer);
	AbstractAuthenticationToken token = this.jwtAuthenticationConverter.convert(jwt);
	token.setDetails(bearer.getDetails());
	return token;
}

```

결국 이곳에서 getJwt 함수를 호출해서 JWT 타입의 객체를 만들고 그것을 시큐리티 컨텍스트에서 쓸 수 있는 토큰으로 만들어서 return 을 하게 됩니다 
getJwt 는 하단에서 바로 볼 수 있다 싶히 `return this.jwtDecoder.decode(bearer.getToken());` 에서 decode 하는 함수를 사용하게 되는데 이때 jwtDecoder
우리가 앞에서 보았듯이 NimbusJwtDecode 가 선택이 될것이다 이 부분은 생략은 했지만 실제로 리소스 서버를 기동할때 이 NimbusJwtDecode 를 기본정책으로 채택을 하게 됩니다 

## NimbusJwtDecode 
```

@Override
public Jwt decode(String token) throws JwtException {
	JWT jwt = parse(token);
	
	...

	Jwt createdJwt = createJwt(token, jwt);
	return validateJwt(createdJwt);
}

```

이 부분에서 parse 를 호출하게 되면 JWT 특정 . 을 기준으로 헤더 , 페이로드 , 서명 이렇게 분리해서 나오게 되고 그게 JWT 타입의 객체로 만들어지는데 이때 이 JWT 타입이라는 것을 각각 디코딩해서 클레임셋이 만들어지는 경우입니다 

```
{"kid":"dp7lpFYPY-do8i9U6Vp3sqb4atyutsw1DUQtZZiwI_s","typ":"JWT","alg":"RS256"}

{"exp":1700910115,"iat":1700909815,"auth_time":1700909792,"jti":"38b79217-7b38-4ffb-be84-58c63adde49c","iss":"http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project","aud":"account","sub":"30283228-fa36-4d88-82c3-c9494dc44bfd","typ":"Bearer","azp":"Spring-Oauth2-Authorizaion-client","session_state":"f420fc5c-e5ba-488c-a905-0331b3fe7901","acr":"1","realm_access":{"roles":["offline_access","uma_authorization","default-roles-srping-oauth2-authorizaion-project"]},"resource_access":{"account":{"roles":["manage-account","manage-account-links","view-profile"]}},"scope":"email profile","sid":"f420fc5c-e5ba-488c-a905-0331b3fe7901","email_verified":false,"name":"time user","preferred_username":"user1","given_name":"time","family_name":"user","email":"user1@gmail.com"}

[서명 ~~~~]

```

이렇게 헤더 , 페이로드 시그니쳐 이렇게 분리가 된 상태입니다 JWT 형태에 대해서는 여러번 언급도 했으니 설명은 넘어가게 되고 이 토큰이 유효한지 아닌지는 제일 마지막 인코딩 되지 않은 
서명부분에 달려 있습니다 그리고 이 서명부분을 검증할떄는 KeyClock 에 주는 기본적인 access_token 은 기본적으로 RSA - 256 타입의 비대칭키입니다 그럼 비대칭키에 대한 
공개키를 가지고 와햐 하는데 `createJwt(token, jwt)` 함수를 호출할때 

## RemoteJWKSet
```
private JWKSet updateJWKSetFromURL() throws RemoteKeySourceException {
	Resource res;
	try {
		res = jwkSetRetriever.retrieveResource(jwkSetURL);
	} catch (IOException e) {
		throw new RemoteKeySourceException("Couldn't retrieve remote JWK set: " + e.getMessage(), e);
	}
	JWKSet jwkSet;
	try {
		jwkSet = JWKSet.parse(res.getContent());
	} catch (java.text.ParseException e) {
		throw new RemoteKeySourceException("Couldn't parse remote JWK set: " + e.getMessage(), e);
	}
	jwkSetCache.put(jwkSet);
	return jwkSet;
}

```
JWKSet 타입의 함수를 호출하게 되는데 JWK 는 (Json Web Key) 이고 이게 비대칭키의 공개키라고 생각하면됩니다 그리고 set 타입이기 때문에 여러개의 공개키가 넘어오게 되는데 
이때 이 부분이 인가서버랑 통신을 하게 되는 부분입니다 retrieveResource  jwkSetURL 를 호출하게 되면 공개키 여러개가 return 이 됩니다 이때 
jwkSetURL 은 우리가 앞의 application.properties 를 에서 적어둔 jwkSetUri 또는 issuer-uri -> jwkSetUri 를 찾는 방식으로 주소를 가저오게 됩니다 

그리고 이 http 통신을 통해서 넘어오는 공개키는 

```

jwkSet = {JWKSet@7668} 
...

keys = {Collections$UnmodifiableList@7674}  size = 2
customMembers = {Collections$UnmodifiableMap@7675}  size = 0

```
2개의 공개key 가 넘어오게 됩니다 

## DefaultJWTProcessor

```

public JWTClaimsSet process(final SignedJWT signedJWT, final C context) {

	...

	List<? extends Key> keyCandidates = selectKeys(signedJWT.getHeader(), claimsSet, context);

	...

	ListIterator<? extends Key> it = keyCandidates.listIterator();

	while (it.hasNext()) {

			JWSVerifier verifier = getJWSVerifierFactory().createJWSVerifier(signedJWT.getHeader(), it.next());

			...

			final boolean validSignature = signedJWT.verify(verifier);

			if (validSignature) {
				return verifyClaims(claimsSet, context);
			}

	}


	...
}

```

selectKeys(signedJWT.getHeader(), claimsSet, context); 함수의 호출과 동시에 위에서 보았던 http 통신을 통한 공개키를 가져오게 되고 그중에서 이 서명을 verify 할 수 있는 공개key 를 찾아서 다음으로 함수로 넘어가게 됩니다 그리고 여러개의 공개key 중에서 verify 함수의 true 가 반환되는 공개키가 있는지 계속 탐색을 하게 됩니다 
이때 true 가 되면 이제 claimsSet 을 반환하게 되는데 이는 JWT 타입의 모든 claimsSet 을 반환하게 됩니다 


그럼 인증이 끝나게 되어서 JWT의 클레임셋을 분리해서 시큐리티 컨텍스트에 저장할 수 있는 모양으로 만듭니다 

```

authenticationResult = {JwtAuthenticationToken@7880} "JwtAuthenticationToken [Principal=org.springframework.security.oauth2.jwt.Jwt@6cc8a9e8, Credentials=[PROTECTED], Authenticated=true, Details=WebAuthenticationDetails [RemoteIpAddress=0:0:0:0:0:0:0:1, SessionId=null], Granted Authorities=[SCOPE_email, SCOPE_profile]]"
 name = "30283228-fa36-4d88-82c3-c9494dc44bfd"
 principal = {Jwt@7878} 
 credentials = {Jwt@7878} 
 token = {Jwt@7878} 
 authorities = {Collections$UnmodifiableRandomAccessList@7886}  size = 2
 details = {WebAuthenticationDetails@7887} "WebAuthenticationDetails [RemoteIpAddress=0:0:0:0:0:0:0:1, SessionId=null]"
 authenticated = true

```

이런 정보는 클레임셋안에 있고 verify true 로 끝나게 되면 이렇게 분리해서 다시 BearerTokenAuthenticationFilter 으로 넘어오게 됩니다 

```

AuthenticationManager authenticationManager = this.authenticationManagerResolver.resolve(request);
Authentication authenticationResult = authenticationManager.authenticate(authenticationRequest);

SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authenticationResult);
SecurityContextHolder.setContext(context);
this.securityContextRepository.saveContext(context, request, response);

```
이 부분을호 넘어오게 됩니다 그리고 그 데이터를 저장하고 끝이나게 되는것이죠 그러면 리소스 서버는 이제 인증이 완료 되었고 최초인증으로 리다이렉트해서 진행을 하게 됩니다 
여기 까지 우리는 리소스 서버가 어떻게 검증을 진행을 할때 어떤 필터를 거치는지에 대해서 공부를 해보았습니다 