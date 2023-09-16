---

title: Spring Secuirty 5 인증 전체과정
author: kimdongy1000
date: 2023-05-23 13:20
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

우리는 지난시간까지 인증이 없는 유저에 대해 시큐리티가 어떻게 인증페이지로 유도하는지 그리고 만들지도 않은 로그인 로그아웃 페이지를 어떻게 만들어내는지에 대해서 살펴보았다 이제 우리는 로그인할때 어떤 일이 일어나는지에 대해서 알아볼려고 합니다 

로그인에 관련한 필터는 UsernamePasswordAuthenticationFilter 이 기본이 되는데 여기서 부터는 조금 filter 의 내용이 길어집니다 그럴 수 밖에 없는게 진짜 로그인을 하고 안에 권한 부여 및 유저를 확인하는 작업을 진행하기 떄문입니다 천천히 가도록 하겠습니다 

## UsernamePasswordAuthenticationFilter

```
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter 


public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException {
		if (this.postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
		}
		String username = obtainUsername(request);
		username = (username != null) ? username.trim() : "";
		String password = obtainPassword(request);
		password = (password != null) ? password : "";
		UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,
				password);
		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);
		return this.getAuthenticationManager().authenticate(authRequest);
	}


```

우리는 시큐리티가 주는 기본 로그인 아이디와 비밀번호로 로그인을 하게 되면 

```
if (this.postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
		}

```
이 부분을 마딱드리게 되는데 이때 로그인의 요청은 항상 post 로그인이야 한다 그렇지 않으면 에러를 던지며 끝이 나게 된다 

```

String username = obtainUsername(request);
		username = (username != null) ? username.trim() : "";
		String password = obtainPassword(request);
		password = (password != null) ? password : "

```

지금보면 이제 요청정보에서 username , password 를 꺼내는 모습을 볼 수 있다 

```
sernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,password);

```
어찌보면 제일 중요한 부분이다 sernamePasswordAuthenticationToken 이 앞으로 이 인증의 유무가 갈리게 되는것인데 

```

public static UsernamePasswordAuthenticationToken unauthenticated(Object principal, Object credentials) {
		return new UsernamePasswordAuthenticationToken(principal, credentials);
	}

	public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
		super(null);
		this.principal = principal;
		this.credentials = credentials;
		setAuthenticated(false);
	}

	public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
		Assert.isTrue(!isAuthenticated,
				"Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
		super.setAuthenticated(false);
	}


```
이 작업을 보면 앞에서 넘어오는 username , password 를 UsernamePasswordAuthenticationToken 객체에 있는 principal , credentials 에 각각 심는것을 보여준다 
다만 이때 유의해서 봐야 할것은 setAuthenticated 이다 인증여부를  true , false 로 다루는것이기에 일단 false 로 두고 인증이 되지 않은 채로 false 로만 두고 토큰을 반환한다 그래서 여기까지 온 token 값을 살펴보면 

```
authRequest = {UsernamePasswordAuthenticationToken@7299} "UsernamePasswordAuthenticationToken [Principal=user, Credentials=[PROTECTED], Authenticated=false, Details=null, Granted Authorities=[]]"
 principal = "user"
 credentials = "08877e55-bd37-43d9-9bc6-1e52495631d0"
 authorities = {Collections$EmptyList@7300}  size = 0
 details = null
 authenticated = false

```

앞에서 넘겨준 username , credentials 만 존재하고 나머지는 아직 값이 없고 authenticated false 인것을 볼 수 있습니다 

그 다음이는 return 을 해주게 되는데 

```

return this.getAuthenticationManager().authenticate(authRequest);

```


## ProviderManager

```

public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean 



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

지금보면 이제 다른 클래스로 넘어온것을 볼 수 있습니다 ProviderManager UsernamePasswordAuthenticationFilter 가 하는 역활은 요청에서 아이디 비밀번호를 추출해서
`UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,password);` 미인가 토큰만 만들고 
실제 이런 토큰에 권한과 인증여부를 판별하는 곳은 ProviderManager 가 되는것입니다 

디버깅을 조금 내려오면

```
try {
	parentResult = this.parent.authenticate(authentication);
	result = parentResult;
}

```
이 부분이 보일것입니다 그런데 좀 이상하게 보입니다 다시 한번 자신의 부모의 authenticate 함수를 호출하게 됩니다 그러면 자기자신이 구현체이기 때문에 
위의 함수가 한번더 호출이 됩니다 그런데 이번에는 

`result = provider.authenticate(authentication);` 호출이 들어가게 되고 이때 이 호출되는 부분은 



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
			catch (UsernameNotFoundException ex) {
				this.logger.debug("Failed to find user '" + username + "'");
				if (!this.hideUserNotFoundExceptions) {
					throw ex;
				}
				throw new BadCredentialsException(this.messages
						.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
			}
			Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
		}
		try {
			this.preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		}
		catch (AuthenticationException ex) {
			if (!cacheWasUsed) {
				throw ex;
			}
			// There was a problem, so try again after checking
			// we're using latest data (i.e. not from the cache)
			cacheWasUsed = false;
			user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			this.preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		}
		this.postAuthenticationChecks.check(user);
		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}
		Object principalToReturn = user;
		if (this.forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}
		return createSuccessAuthentication(principalToReturn, authentication, user);
	}

```

```

String username = determineUsername(authentication);
boolean cacheWasUsed = true;
UserDetails user = this.userCache.getUserFromCache(username);

```
지금보면 넘어오는 인증객체의 username 을 추츨해서 UserDetails 라는 객체를 만들고 있습니다 이 UserDetails 또한 시큐리티에서 중요한 인터페이스 입니다 실제로 모든 유저들은 이 UserDetails 구현체에 담기게 됩니다 그리고 `user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);` 부분에서 
실제 user 의 정보가 담기게 되는데 아래로 따라가면 


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
	catch (UsernameNotFoundException ex) {
		mitigateAgainstTimingAttack(authentication);
		throw ex;
	}
	catch (InternalAuthenticationServiceException ex) {
		throw ex;
	}
	catch (Exception ex) {
		throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
	}
}


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

다시되돌아오게 되면 

```
user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);

```
이제 이 user 객체 안에는 

```
user = {User@7824} "org.springframework.security.core.userdetails.User [Username=user, Password=[PROTECTED], Enabled=true, AccountNonExpired=true, credentialsNonExpired=true, AccountNonLocked=true, Granted Authorities=[]]"
 password = "{bcrypt}$2a$10$TGqvEOp2TW98azOTgkejbOhbEkepWtD8p8ushIcXeDnjBxO1KCcWS"
 username = "user"
 authorities = {Collections$UnmodifiableSet@7836}  size = 0
 accountNonExpired = true
 accountNonLocked = true
 credentialsNonExpired = true
 enabled = true

```
이런 데이터가 들어오게 됩니다 그럼 끝이냐 아닙니다 아직 비밀번호 검증을 안했기 때문에 이제 그 검증을 해야 합니다 바로 하단 

```
try {
	this.preAuthenticationChecks.check(user);
	additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
}

```
추가적인 검증으로 additionalAuthenticationChecks 로 넘기게 됩니다 

## DaoAuthenticationProvider
```

public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider 


@Override
@SuppressWarnings("deprecation")
protected void additionalAuthenticationChecks(UserDetails userDetails,
		UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
	if (authentication.getCredentials() == null) {
		this.logger.debug("Failed to authenticate since no credentials provided");
		throw new BadCredentialsException(this.messages
				.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
	}
	String presentedPassword = authentication.getCredentials().toString();
	if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
		this.logger.debug("Failed to authenticate since password does not match stored value");
		throw new BadCredentialsException(this.messages
				.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
	}
}

```
이제 비밀번호 검증을 위해서 이쪽으로 넘어오게 됩니다 

```
if (authentication.getCredentials() == null) {
	this.logger.debug("Failed to authenticate since no credentials provided");
	throw new BadCredentialsException(this.messages
			.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
}

```

여기서 username pricipal 로 쓰였다면 비밀번호는 credentials 로 쓰이게 됩니다 만약 비밀번호가 넘어오지 않았다면 BadCredentialsException 을 던지고 끝이나게 됩니다

```

String presentedPassword = authentication.getCredentials().toString();
		if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			this.logger.debug("Failed to authenticate since password does not match stored value");
			throw new BadCredentialsException(this.messages
					.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
		}

```
하지만 비밀번호가 있다면 그것을 꺼내고 presentedPassword form 에서 넘오는 비밀번호 

this.passwordEncoder.matches(presentedPassword, userDetails.getPassword()) 그리고 loadByUsername 을 했을때 넘어오는 저장된 비밀번호와 passwordEncoder 로 
매핑을 시켜서 일치하는지 여부를 판단합니다 올바르면 이제 추가 인증까지 완료가 된것입니다 그것이 아니고 비밀번호가 틀렸다면 역시나 BadCredentialsException 을 던지고 끝이나게 됩니다 

그리고 모든 인증이 완료되었으니 시큐리티 이 인증객체를 자신의 인증컨텍스트에 심어놓고 사용을 해야 비로소 진정한 인증이 끝나게 됩니다 


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
				// return immediately as subclass has indicated that it hasn't completed
				return;
			}
			this.sessionStrategy.onAuthentication(authenticationResult, request, response);
			// Authentication success
			if (this.continueChainBeforeSuccessfulAuthentication) {
				chain.doFilter(request, response);
			}
			successfulAuthentication(request, response, chain, authenticationResult);
		}
		catch (InternalAuthenticationServiceException failed) {
			this.logger.error("An internal error occurred while trying to authenticate the user.", failed);
			unsuccessfulAuthentication(request, response, failed);
		}
		catch (AuthenticationException ex) {
			// Authentication failed
			unsuccessfulAuthentication(request, response, ex);
		}
	}





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

```

이 안에서 successfulAuthentication 로 넘어가게 되면 

```

SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authResult);
SecurityContextHolder.setContext(context);
this.securityContextRepository.saveContext(context, request, response);

```

먼저 인증컨텍스트에 createEmptyContext 빈 컨텍스트 한자리 마련해두고 여기에 setAuthentication 함수를 호출해 인증이 완료된 객체를 심어줍니다 
그렇게 되면 시큐리티 인증이 끝나게 되고 하단 this.successHandler.onAuthenticationSuccess(request, response, authResult); 을 통해서 
인증이 끝난 콜백 함수를 호출하러 가게 됩니다 


정말 길고긴 인증과정입니다 정리를 한번 하도록 하겠습니다 아래는 요청순서대로 간략하게 정리를 하는 부분입니다 

## 정리 및 요약 

1. 사용자가 로그인할려고 아이디 비밀번호를 입력하면 걸리는 처음 걸리는 Filter 는 UsernamePasswordAuthenticationFilter 입니다 이 Filter 는 사용자가의 request 에서 
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



엄청 길다 이 부분이 시큐리티 부분에서 제일 긴부분입니다 처음에는 이해가 안가고 그럴 수 있지만 반복적으로 디버깅을 하면서 어떤 데이터 어떤 객체가 생성되는지 잘 살펴보면 
충분히 이해하실 수 있습니다 이로서 우리는 인증에 필요한 기본 필터를 알아보았습니다 앞으로는 우리가 직접 커스텀하게 유저를 만들거나 , 로그인 페이지를 생성하는 그런 작업을 하면서 시큐리티를 계속해서 공부해나가겠습니다 