---
title: Spring Secuirty 32 Resource Server Entrypoint
author: kimdongy1000
date: 2023-07-10 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

우리는 지난시간에 KeyClock 와 ResourceServer 을 연동을 해보았다 앞에서 OAuth2 로그인시 권한이 없는 유저를 시큐리티가 어떻게 Redirect 했는지 보았는데 Resource - Server 에서도 똑같이 한번 보겠습니다

## OAuth2ResourceServerConfigurer
```
@Override
public void init(H http) {

	...
	
	registerDefaultEntryPoint(http);

	...
}

```
기동을 하게 되면 init 이 호출이 되면서  registerDefaultEntryPoint 를 호출하게 됩니다 

```
private void registerDefaultEntryPoint(H http) {
	...

	exceptionHandling.defaultAuthenticationEntryPointFor(this.authenticationEntryPoint, preferredMatcher);	
}

```

다 생략을 하고 제일 마지막줄에 보면 기본적인 엔트리포인트 전략을 집어넣게 됩니다 this.authenticationEntryPoint 가 BearerTokenAuthenticationEnttyPoint 
이것을 호출하게 하고 권한이 없는 요청은 BearerTokenAuthenticationEnttyPoint 이곳에서 처리할 예정입니다 


```
@Override
public void configure(H http) {
	
	...
	
	filter.setAuthenticationEntryPoint(this.authenticationEntryPoint);
	filter = postProcess(filter);
	http.addFilter(filter);
}
```
configure 은 앞의 init 으로 넘어오는 BearerTokenAuthenticationEnttyPoint filter 에 등록하는 역할을 하게 됩니다 이렇게 되면 모든 요청 건들은 BearerTokenAuthenticationEnttyPoint 에서 권한여부를 체크하게 됩니다 

첫번째 상황은 토큰 없이 그냥 주소창에 localhost:8082 를 넣었을때 입니다 

## BearerTokenAuthenticationEntryPoint
```

@Override
public void commence(HttpServletRequest request, HttpServletResponse response,
		AuthenticationException authException) {
	
	HttpStatus status = HttpStatus.UNAUTHORIZED;
	Map<String, String> parameters = new LinkedHashMap<>();

	...
	
	if (authException instanceof OAuth2AuthenticationException) {
		
		...
	}

	String wwwAuthenticate = computeWWWAuthenticateHeaderValue(parameters);
	response.addHeader(HttpHeaders.WWW_AUTHENTICATE, wwwAuthenticate);
	response.setStatus(status.value());
}


```
이쪽으로 넘어와서 권한이 없는 유저를 되돌리게 되는데 `HttpStatus status = HttpStatus.UNAUTHORIZED;` 으로 요청을 만들어놓은뒤 마지막에 
`response.setStatus(status.value());` 처리를 해서 우리는 헤더에 401을 달아버리고 그냥 끝이나게 됩니다 (리다이렉트 없음)


두번째 상황은 post - man 으로 요청을 줄때 올바르지 않은 토큰을 줄떄입니다 

```
@Override
public void commence(HttpServletRequest request, HttpServletResponse response,
		AuthenticationException authException) {
	HttpStatus status = HttpStatus.UNAUTHORIZED;
	Map<String, String> parameters = new LinkedHashMap<>();
	if (this.realmName != null) {
		parameters.put("realm", this.realmName);
	}
	if (authException instanceof OAuth2AuthenticationException) {
		OAuth2Error error = ((OAuth2AuthenticationException) authException).getError();
		parameters.put("error", error.getErrorCode());
		if (StringUtils.hasText(error.getDescription())) {
			parameters.put("error_description", error.getDescription());
		}
		if (StringUtils.hasText(error.getUri())) {
			parameters.put("error_uri", error.getUri());
		}
		if (error instanceof BearerTokenError) {
			BearerTokenError bearerTokenError = (BearerTokenError) error;
			if (StringUtils.hasText(bearerTokenError.getScope())) {
				parameters.put("scope", bearerTokenError.getScope());
			}
			status = ((BearerTokenError) error).getHttpStatus();
		}
	}
	String wwwAuthenticate = computeWWWAuthenticateHeaderValue(parameters);
	response.addHeader(HttpHeaders.WWW_AUTHENTICATE, wwwAuthenticate);
	response.setStatus(status.value());
	}
```

아까오는 달리 `if (authException instanceof OAuth2AuthenticationException)` 이 부분이 rue 로 들어오게 된다 왜냐하면 앞에서는 토큰없이 들어왔기 때문에 authException false 로 뜨는 반면 지금은 토큰은 있고 토큰을 검증하는 와중에 에러가 발생한 것이기 떄문에 이 if 문 안으로 들어오게 됩니다 

An error occurred while attempting to decode the Jwt: Invalid JWT serialization: Missing dot delimiter(s) 이런 에러가 메세지메 맺히게 되고 뜻은 jwt 의 점 형식이 맞지 않습니다 뒤에 설명할것이지만 넘어오는 토큰은 jwt 형태로 넘어오게 되고 .을 기준으로 나뉘게 되는데 그런 형식 없이 넘어와서 에러를 발생한 상태입니다 

`status = ((BearerTokenError) error).getHttpStatus();` 마지막에서 호출되는 status 또는 401 에러로 발생이 되어서 끝나게 됩니다 

이로서 알 수 있는 사실은 인증이 없거나 또는 유효하지 않은 토큰을 보내게 되면 BearerTokenAuthenticationEntryPoint 에서 처리를 하는 모습을 볼 수 있습니다 
2가지 경우를 보았는데 동일한 401 에러가 발생하는 것을 보았습니다 우리는 앞으로 이곳에서 에러를 한번더 볼 예정입니다 403 에러가 발생될 예정인데 403 에 대해서는 
뒤에 scope 할때 다뤄보겠습니다 오늘은 인증이 없는 유저에 대해서 시큐리티가 어떻게 응답을 주는지에 대해서 알아보았습니다