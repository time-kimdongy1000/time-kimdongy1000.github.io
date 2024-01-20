---
title: Spring Secuirty 5 인증 전체과정
author: kimdongy1000
date: 2023-05-23 13:20
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---
이번시간에는 실제로 로그인을 하시 어떤 일이 일어나는지에 대해서 알아보도록 하겠습니다 

로그인에 관련한 필터는 UsernamePasswordAuthenticationFilter 이 기본이 되는데 여기서 부터는 조금 filter 의 내용이 길어집니다 그럴 수 밖에 없는게 여기서는 실제로 User 가 존재하는지 여부와 사용자 권한 확보 그리고 인증된 객체를 return 을 해야 함으로 Filter 길이가 길어질 수 밖에 없습니다 

## UsernamePasswordAuthenticationFilter
```
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException 
{
	if (this.postOnly && !request.getMethod().equals("POST")) {
		throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
	}
	String username = obtainUsername(request);
	username = (username != null) ? username.trim() : "";
	String password = obtainPassword(request);
	password = (password != null) ? password : "";
	UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,
			password);
	setDetails(request, authRequest);
	return this.getAuthenticationManager().authenticate(authRequest);
}
```

우리는 시큐리티가 주는 기본 로그인 아이디와 비밀번호로 로그인을 하게 되면 

## 요청 메서드 확인
```
if (this.postOnly && !request.getMethod().equals("POST")) {
	throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
}
```

이 분기문으로 들어오게 되는데 이때 로그인 요청은 항상 POST 여야 합니다 그렇지 않으면 예외처리하고 끝이나게 됩니다 


## Form 정보 가져오기 (username , password)
```
String username = obtainUsername(request);
username = (username != null) ? username.trim() : "";
String password = obtainPassword(request);
password = (password != null) ? password : "
```
이 부분에서는 Form 정보에 기입한 username , password 를 요청에서 꺼내오게 됩니다 

## 미인증 토큰 발급 
```

UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,password);

```

제일 중요한 부분이다 UsernamePasswordAuthenticationToken 이 앞으로 이 인증의 유무가 갈리게 되는것인데 지금은 받은 username 하고 password 를 통해서 
UsernamePasswordAuthenticationToken 객체만 만들어 놓았지 실제로 unauthenticated 살펴보면 이는 인증이 되지 않았음을 보여줍니다 이 인증은 ProviderManager 를 통해서 
인증 토큰으로 바뀌게 됩니다 

## ProviderManager
```
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean 
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		...

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
			}	
			.....
		}
		if (result == null && this.parent != null) {
			try {
				parentResult = this.parent.authenticate(authentication);
				result = parentResult;
			}
			...
		}	
	}
```
ProviderManager 는 인증 프로세서를 관리하는 매니저입니다 이 매니저는 다양한 인증 Provider 을 제공하고 해당 요청을 보고 적절한 AuthenticationProvider 로 인도하게 됩니다 
그런 과정은 `int size = this.providers.size();` 이 부분에서 몇개의 Provider 이 제공되는지 체크 후 

```
for (AuthenticationProvider provider : getProviders()) {
		if (!provider.supports(toTest)) {
			continue;
		}
}

```
가장 적절한 인증Provider 로 인도를 하게 됩니다 이떄 채택된 provider 로 아래의 메서드 `result = provider.authenticate(authentication);`  를 호출하게 되는데 

## AbstractUserDetailsAuthenticationProvider

```
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
				() -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));
		String username = determineUsername(authentication);
		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);
		if (user == null) {
			cacheWasUsed = false;
			try {
				user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			}
		...
		return createSuccessAuthentication(principalToReturn, authentication, user);
	}
}

```
AbstractUserDetailsAuthenticationProvider 는 제공된 미인증 객체를 이용해서 `retrieveUser` 에서 사용자를 가져옴 이때는 인메모리 DB 또는 RDBMS 등 특정 데이터를 가지고 
User 객체로 변환할 수 있는 어떤것도 가능합니다 기본적으로 인메모리 유저를 사용하기 때문에 인메모리 데이터베이스에서 가져오게 됩니다 

## DaoAuthenticationProvider
```
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider 

@Override
protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
		throws AuthenticationException {
	prepareTimingAttackProtection();
	try {
		UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
		if (loadedUser == null) {
			throw new InternalAuthenticationServiceException(
					"UserDetailsService returned null, which is an interface contract violation");
		}
		return loadedUser;
	}
	...	
}
이 부분이 `retrieveUser` 호출시 인메모리 데이터 베이스에서 사용자를 가져오는 함수입니다 


private void prepareTimingAttackProtection() {
	if (this.userNotFoundEncodedPassword == null) {
		this.userNotFoundEncodedPassword = this.passwordEncoder.encode(USER_NOT_FOUND_PASSWORD);
	}
}


```

`prepareTimingAttackProtection` 함수 호출시 패스워드에 인코딩이 안걸려 있으면 에러 던지고 끝이 납니다 그리고 비밀번호가 인코딩되어서 들어왔다면 이제 진짜 이 유저가 존재하는지 안하는지 찾게 됩니다 그것이 아래에 있는 loadUserByUsername 이 됩니다 즉 username 을 먼저 가지고 유저가 진짜 존재하는지 안하는지 검색을 먼저 합니다 
`UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);` 


## InMemoryUserDetailsManager
```
public class InMemoryUserDetailsManager implements UserDetailsManager, UserDetailsPasswordService

public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
		UserDetails user = this.users.get(username.toLowerCase());
		if (user == null) {
			throw new UsernameNotFoundException(username);
		}
		return new User(user.getUsername(), user.getPassword(), user.isEnabled(), user.isAccountNonExpired(),
				user.isCredentialsNonExpired(), user.isAccountNonLocked(), user.getAuthorities());
	}

```
그럼 시큐리티는 최초 유저를 어디다가 저장을 해놓느냐 바로 InMemoryUserDetailsManager 즉 메모리에 유저를 저장을 하고 그것을 검색해서 가져다 쓰게 됩니다 
`UserDetails user = this.users.get(username.toLowerCase());` 이를 통해서 `private final Map<String, MutableUserDetails> users = new HashMap<>();`
여기 HashMap 에 담겨 있는 user 를 검색해서 가져오게 됩니다 만약 없으면 UsernameNotFoundException 던지고 끝이나고 그게 아니라면 이제 이 진짜 user 의 데이터를 심어주게 됩니다 

그러면 다시 AbstractUserDetailsAuthenticationProvider 되돌아 옵니다 이제 유저도 찾았으니 모두 return 을 받은것이지요 
`user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);` 그러면 이 user 에는 일단 찾은 User 의 객체가 있지만 이것으로 끝은 아닙니다 
다음은 비밀번호 검증을 진행을 해야 합니다 


## DaoAuthenticationProvider
```
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider 

@Override
@SuppressWarnings("deprecation")
protected void additionalAuthenticationChecks(UserDetails userDetails,
		UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
	if (authentication.getCredentials() == null) {
		this.logger.debug("Failed to authenticate since no credentials provided");
		throw new BadCredentialsException(this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
	}
	String presentedPassword = authentication.getCredentials().toString();
	if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
		this.logger.debug("Failed to authenticate since password does not match stored value");
		throw new BadCredentialsException(this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
	}
}

```

```
if (authentication.getCredentials() == null) {
	this.logger.debug("Failed to authenticate since no credentials provided");
	throw new BadCredentialsException(this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
}

```

여기서 username pricipal 로 쓰였다면 비밀번호는 credentials 로 쓰이게 됩니다 만약 비밀번호가 넘어오지 않았다면 BadCredentialsException 을 던지고 끝이나게 됩니다

```

String presentedPassword = authentication.getCredentials().toString();
if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
	this.logger.debug("Failed to authenticate since password does not match stored value");
	throw new BadCredentialsException(this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
}

```
하지만 비밀번호가 있다면 그것을 꺼내고 presentedPassword form 에서 넘오는 비밀번호 

this.passwordEncoder.matches(presentedPassword, userDetails.getPassword()) 그리고 loadByUsername 을 했을때 넘어오는 저장된 비밀번호와 passwordEncoder 로 
매핑을 시켜서 일치하는지 여부를 판단합니다 올바르면 이제 추가 인증까지 완료가 된것입니다 그것이 아니고 비밀번호가 틀렸다면 역시나 BadCredentialsException 을 던지고 끝이나게 됩니다 

이제 인증은 끝이났고 이를 SecurityContext 에 setAuthentication 을 하게 되면 인증은 끝이나게 됩니다 


## AbstractAuthenticationProcessingFilter
```
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
		implements ApplicationEventPublisherAware, MessageSourceAware 

	private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		if (!requiresAuthentication(request, response)) {
			chain.doFilter(request, response);
			return;
		}
		try {

			Authentication authenticationResult = attemptAuthentication(request, response);
			if (authenticationResult == null) {
				return;
			}

			this.sessionStrategy.onAuthentication(authenticationResult, request, response);

			if (this.continueChainBeforeSuccessfulAuthentication) {
				chain.doFilter(request, response);
			}

			successfulAuthentication(request, response, chain, authenticationResult);
		}
		...	
	}

	protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
			Authentication authResult) throws IOException, ServletException {
		SecurityContext context = SecurityContextHolder.createEmptyContext();
		context.setAuthentication(authResult);
		SecurityContextHolder.setContext(context);
		this.securityContextRepository.saveContext(context, request, response);

		...
		
		this.successHandler.onAuthenticationSuccess(request, response, authResult);
	}

```
이 부분에서 앞에서 인증을 점겨준 정보를 가지고 마지막으로 SecurityContext 에 인증정보를 심는 Filter 입니다 


```
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authResult);
SecurityContextHolder.setContext(context);
this.securityContextRepository.saveContext(context, request, response);

```

먼저 인증컨텍스트에 createEmptyContext 빈 컨텍스트 한자리 마련해두고 여기에 setAuthentication 함수를 호출해 인증이 완료된 객체를 심어줍니다 
그렇게 되면 시큐리티 인증이 끝나게 되고 하단 this.successHandler.onAuthenticationSuccess(request, response, authResult); 을 통해서 
인증이 끝난 콜백 함수를 호출하러 가게 됩니다 그러면 시큐리티의 인증은 끝이나게 됩니다 


## 정리 및 요약 

1. 사용자가 로그인할려고 아이디 비밀번호를 입력하면 걸리는 처음 걸리는 Filter 는 UsernamePasswordAuthenticationFilter 입니다 이 Filter 는 사용자의 request 에서 
username , password 를 뽑아내어서 아직 인증이 되지 않은  UsernamePasswordAuthenticationToken 객체를 생성하고 인증은 
return this.getAuthenticationManager().authenticate(authRequest); 함수를 통해 ProviderManager 에게 넘겨줍니다 

2. ProviderManager 는 UsernamePasswordAuthenticationFilter 넘겨준 아이디 비밀번호만 들어가 있는 미인증 객체를 활용해서 이 미인증 객체를 인증객체로 만드는 역활을 합니다 

3. DaoAuthenticationProvider 는 먼저 이 user 가 존재하는지 안하는지 오로지 username 으로만 user 를 검색한다 우리는 직접 user를 생성하지 않았고 
기본적으로 만들어진 user 를 사용하기 때문에 InMemoryUserDetailsManager 에서 유저를 가져올텐데 이는 메모리에 저장된 user 를 가져오게 됩니다 

4. UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username); 으로 return new User(user.getUsername(), user.getPassword(), user.isEnabled(), user.isAccountNonExpired(), user.isCredentialsNonExpired(), user.isAccountNonLocked(), user.getAuthorities()); 호출을 통해서 저장된 비밀번호 유저의 상태 그리고 권한까지 삽입을 하고 return 을 합니다 

5. 우리는 4번까지 아이디로만 검색해서 user가 존재하는지 아닌지만 검색을 했습니다 이제 비밀번호 검증을 해야 합니다 그에 필요한 작업은 역시나 
DaoAuthenticationProvider 에서 진행을 하게 됩니다 

6. 5번 작업으로 인해서 인증이 완벽하게 되었으면 이제 시큐리티는 이 정보를 가지고 SecurityContext 에 인증정보를 심게 됩니다 이로서 인증은 마무리 되고
인증이 완료된 직후 행동을 onAuthenticationSuccess 를 호출하게 됩니다

이것이 시큐리티에서 제공하는 가장 기본적인 로그인 및 인증 방식입니다 