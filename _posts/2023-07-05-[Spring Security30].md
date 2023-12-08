---
title: Spring Secuirty 30 OIDC 로그아웃
author: kimdongy1000
date: 2023-07-05 18:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

지난시간에 OIDC 인증에 관련한 Flow 를 살펴보았습니다 잠깐 복습을 해보자면 OIDC 는 인증에 관련한 프레임워크로 OAuth2.0 위의 계층에 존재하는 프레임워크입니다 
access_token , refres_token 은 인가쪽인 부분임으로 이 부분은 로그아웃이 따로 존재하지는 않지만 인증에 대한 OIDC 는 로그아웃이 존재하기 때문에 
이 부분을 다루도록 하겠습니다 

## Security 로그아웃 
사실 시큐리티 로그아웃은 간단합니다 Security 컨텍스트에 저장되어 있는 정보를 삭제 (세선무효화 쿠키 삭제) 등을 하고 리딕렉션을 처음 로그인 페이지로 보내게 되면 
사실상 클라이언트 상에서는 OIDC 로그아웃이나 , OAuth2 로그아웃이나 동일하게 보일 예정입니다 시큐리티 로그아웃은 OAuth2 , OIDC 나 상관없이 
모두 LogoutFilter 를 타고 이 필터의 결과가 

```
SecurityContext context = SecurityContextHolder.getContext();
SecurityContextHolder.clearContext();
if (this.clearAuthentication) {
	context.setAuthentication(null);
}
```

이렇게 컨텍스를 지우는것으로 끝이나기 때문입니다 그럼 굳이 OIDC 로그아웃을 하는 이유는 이는 클라이언트 로그아웃 뿐만 아니라 KeyClock 에 있는 세션정보까지도 없애기 위한 로그아웃입니다 우리 다른 어플리케이션 보면 모든기기에서 로그아웃 하기 이런 기능을 본적이 있습니다 이 버튼을 누르면 내가 인증 인가를 했던 모든기기에서 로그아웃이 진행이 되기 때문에 이 코드를 동작시키면 우리는 다시 로그인을 해야하는 상황이 오기 때문입니다 

로그인을 한 상태에서 `http://localhost:8081/logout` 이렇게 로그아웃 요청을 주게 되면 

![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/021f1490-2a91-483d-9cf9-3d81724b96af)

이렇게 우리 앞에서 보았던 Form 로그아웃 처럼 기본적인 로그아웃 페이지가 뜨게 되고 로그아웃 버튼을 누르게 되면


## LogoutFilter 
```
public class LogoutFilter extends GenericFilterBean {

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);
	}

	private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		if (requiresLogout(request, response)) {
			Authentication auth = SecurityContextHolder.getContext().getAuthentication();
			if (this.logger.isDebugEnabled()) {
				this.logger.debug(LogMessage.format("Logging out [%s]", auth));
			}
			this.handler.logout(request, response, auth);
			this.logoutSuccessHandler.onLogoutSuccess(request, response, auth);
			return;
		}
		chain.doFilter(request, response);
	}
}

private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
		throws IOException, ServletException {
	if (requiresLogout(request, response)) {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		if (this.logger.isDebugEnabled()) {
			this.logger.debug(LogMessage.format("Logging out [%s]", auth));
		}
		this.handler.logout(request, response, auth);
		this.logoutSuccessHandler.onLogoutSuccess(request, response, auth);
		return;
	}
	chain.doFilter(request, response);
}
```

이렇게 LogoutFilter 가 동작을 하게 됩니다 
`Authentication auth = SecurityContextHolder.getContext().getAuthentication();` 현재 시큐리티 컨텍스트 에 저장된 인증객체를 가져오고 
`this.handler.logout(request, response, auth);` 호출을 통해서 로그아웃을 시도하게 됩니다 


## CompositeLogoutHandler
```
@Override
public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
	for (LogoutHandler handler : this.logoutHandlers) {
		handler.logout(request, response, authentication);
	}
}
```

이쪽으로 호출이 되는데 이때 handler 3개의 핸들러가 저장이 되어 있고 이를 3번 호출하면서 각 핸들러마다 로그아웃을 진행하게 됩니다 

3개의 핸들러는 

```
logoutHandlers = {Arrays$ArrayList}  size = 3
 0 = {CsrfLogoutHandler} 
  csrfTokenRepository = {LazyCsrfTokenRepository} 
 1 = {SecurityContextLogoutHandler} 
  logger = {LogAdapter$Slf4jLocationAwareLog} 
  invalidateHttpSession = true
  clearAuthentication = true
 2 = {LogoutSuccessEventPublishingLogoutHandler} 
```

CsrfLogoutHandler 
SecurityContextLogoutHandler
LogoutSuccessEventPublishingLogoutHandler

이렇게 총 3개의 핸들러에서 로그아웃을 호출하게 되는데 하나씩 들어가보면 

## CsrfLogoutHandler
```
public final class CsrfLogoutHandler implements LogoutHandler {

	@Override
	public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
		this.csrfTokenRepository.saveToken(null, request, response);
	}
}
```
이쪽으로 들어와서 saveToken 토큰 함수를 호출하게 되는데 함수명만 보면 무엇인가 저장하는거 처럼 보이지만 다음 메서드 호출로 가보자 


## LazyCsrfTokenRepository
```

public final class LazyCsrfTokenRepository implements CsrfTokenRepository {

	@Override
	public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
		if (token == null) {
			this.delegate.saveToken(token, request, response);
		}
	}
}

```
이떄 token 은 null 이고 이떄 token 은 우리가 알고 있는 access_token 이나 , refresh 토큰이 아니라 csrf_token 임을 헷갈리면 안됩니다 
this.delegate.saveToken(token, request, response);


## HttpSessionCsrfTokenRepository
```
public final class HttpSessionCsrfTokenRepository implements CsrfTokenRepository {

	@Override
	public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
		if (token == null) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				session.removeAttribute(this.sessionAttributeName);
			}
		}
		else {
			HttpSession session = request.getSession();
			session.setAttribute(this.sessionAttributeName, token);
		}
	}
}

```
이곳에서 `HttpSession session = request.getSession(false);` 현재 저장된 http 세션을 가져오고 그곳에서 
`session.removeAttribute(this.sessionAttributeName);` `속성을 제거하는데 org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository.CSRF_TOKEN`
보이는 값처럼 CSRF_TOKEN 을 제거하게 됩니다 그럼 첫번째 로그아웃 완료 

## SecurityContextLogoutHandler 
```
public class SecurityContextLogoutHandler implements LogoutHandler {

	@Override
	public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
		Assert.notNull(request, "HttpServletRequest required");
		if (this.invalidateHttpSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				session.invalidate();
				if (this.logger.isDebugEnabled()) {
					this.logger.debug(LogMessage.format("Invalidated session %s", session.getId()));
				}
			}
		}
		SecurityContext context = SecurityContextHolder.getContext();
		SecurityContextHolder.clearContext();
		if (this.clearAuthentication) {
			context.setAuthentication(null);
		}
	}
}
```

이 부분이 사실상 핵심인 부분이다 session != null 이 아니므로 들어와서 session.invalidate(); 호출해서 세션을 무효화 시키고 

```

SecurityContext context = SecurityContextHolder.getContext();
SecurityContextHolder.clearContext();

if (this.clearAuthentication) {
		context.setAuthentication(null);
	}

```

이 부분 호출을 통해서 인증 컨텍스를 깨끗하게 지워버리게 된다 `SecurityContextHolder.clearContext();` 호출함으로서 컨텍스를 지우겠다는 뜻으로 받아들이고 
실제 인증정보는 `context.setAuthentication(null);` 이곳을 호출함으로서 지워지게 됩니다 

```
context = {SecurityContextImpl@8431} "SecurityContextImpl [Null authentication]"
authentication = null
```

호출이 되고 나면 이 컨텍스트에는 아무런 값이 남지 않게 됩니다 


## LogoutSuccessEventPublishingLogoutHandler 

```
public final class LogoutSuccessEventPublishingLogoutHandler implements LogoutHandler, ApplicationEventPublisherAware {

	@Override
	public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
		if (this.eventPublisher == null) {
			return;
		}
		if (authentication == null) {
			return;
		}
		this.eventPublisher.publishEvent(new LogoutSuccessEvent(authentication));
	}
}

```
얘는 좀 생소할 수 있는데 로그아웃이 성공적으로 되었을때 발생되는 이벤트를 이곳에서 진행이 됩니다 크게 볼것은 없으니 넘어가도록 하겠습니다 

## AbstractAuthenticationTargetUrlRequestHandler
```

public abstract class AbstractAuthenticationTargetUrlRequestHandler {

	protected void handle(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
			throws IOException, ServletException {
		String targetUrl = determineTargetUrl(request, response, authentication);
		if (response.isCommitted()) {
			this.logger.debug(LogMessage.format("Did not redirect to %s since response already committed.", targetUrl));
			return;
		}
		this.redirectStrategy.sendRedirect(request, response, targetUrl);
	}
}

```
이곳에서 targetUrl /login?logout 값이 매겨지게 되고 this.redirectStrategy.sendRedirect(request, response, targetUrl); 통해서 이제 완벽하게 로그아웃 페이지로 인도를 하게 됩니다 

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/9bc9e1aa-3efc-480d-b157-618dc51e8f2f)

그러면 이제 이렇게 로그아웃 성공메세지가 뜨고 바깥으로 나온것을 알 수 있는데 사실 이 부분에서는 OIDC 로그아웃은 동작하지 않았습니다 

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/b765be5b-f75a-4021-83e1-ba6bceff2801)

keyClock 로 들어가보면 아직 세션 정보가 남아 있는것을 확인할 수 있고 OIDC 로그아웃을 하게 되면 이 세션정보도 말끔히 없어지게 될것입다 여기 까지는 시큐리티 상의 로그아웃이 되는것입니다 

## OIDC 로그아웃 

```
@Configuration
public class SecurityConfig {

    @Autowired
    private ClientRegistrationRepository clientRegistrationRepository;


    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

        httpSecurity.authorizeRequests().anyRequest().authenticated();
        httpSecurity.oauth2Login();

        httpSecurity.logout().logoutSuccessHandler(oidcLogoutSuccessHandler());

        return httpSecurity.build();
    }


    public LogoutSuccessHandler oidcLogoutSuccessHandler(){


        OidcClientInitiatedLogoutSuccessHandler oidcClientInitiatedLogoutSuccessHandler = new OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository);
        oidcClientInitiatedLogoutSuccessHandler.setPostLogoutRedirectUri("http://localhost:8081/login");

        return oidcClientInitiatedLogoutSuccessHandler;
    }
}


```

SecurrityConfig 에 위와 같이 작성을 하면 됩니다 `httpSecurity.logout().logoutSuccessHandler(oidcLogoutSuccessHandler());` 시큐리티의 로그아웃이 완벽하게 끝이나게 되면 logoutSuccessHandler 를 호출해서 oidcLogoutSuccessHandler() 호출하게 만들것입니다 그러면 이 부분에서 인가서버에 접근을 해서 세션을 삭제하게 됩니다


![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ec371c78-999c-452a-8a62-0a4443455bf6)

마찬가지로 인가서버에서 되돌아오는것이기 때문에 redirect-url 을 한번더 이번에는 Vaild post logout redirect URLs 에 http://localhost:8081/login 호출하는 주소를 적어주면 됩니다 그러면 

앞의 내용 (csrf , 시큐리티 컨텍스트 지우기는 진행을 하고) 

```

this.logoutSuccessHandler.onLogoutSuccess(request, response, auth);

```

호출하게 되는데 이 부분에서 


## OidcClientInitiatedLogoutSuccessHandler
```
public final class OidcClientInitiatedLogoutSuccessHandler extends SimpleUrlLogoutSuccessHandler {

	@Override
	protected String determineTargetUrl(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) {
		String targetUrl = null;
		if (authentication instanceof OAuth2AuthenticationToken && authentication.getPrincipal() instanceof OidcUser) {
			String registrationId = ((OAuth2AuthenticationToken) authentication).getAuthorizedClientRegistrationId();
			ClientRegistration clientRegistration = this.clientRegistrationRepository
					.findByRegistrationId(registrationId);
			URI endSessionEndpoint = this.endSessionEndpoint(clientRegistration);
			if (endSessionEndpoint != null) {
				String idToken = idToken(authentication);
				String postLogoutRedirectUri = postLogoutRedirectUri(request, clientRegistration);
				targetUrl = endpointUri(endSessionEndpoint, idToken, postLogoutRedirectUri);
			}
		}
		return (targetUrl != null) ? targetUrl : super.determineTargetUrl(request, response);
	}
}


```

이쪽으로 들어오게 됩니다 시큐리티 로그아웃은 완료 되었고 우리가 로그아웃이 완료되면 이곳으로 호출하라고 지시를 했기 때문에 이곳으로 들어 오게 됩니다 

`if (authentication instanceof OAuth2AuthenticationToken && authentication.getPrincipal() instanceof OidcUser)` 현재 인증이 타입이 OidcUser 라면 
인증객체 타입이 OAuth2AuthenticationToken 이면서 OidcUser 이여야 하기 때문에 openid 로 들어오는것에 대해서는 이 분이 false 로 떨어지게 됩니다 


```
String registrationId = ((OAuth2AuthenticationToken) authentication).getAuthorizedClientRegistrationId();
ClientRegistration clientRegistration = this.clientRegistrationRepository.findByRegistrationId(registrationId);
```
현재 사용중인 ClientRegistration 가져오게 되고 


```
URI endSessionEndpoint = this.endSessionEndpoint(clientRegistration);
			if (endSessionEndpoint != null) {
				String idToken = idToken(authentication);
				String postLogoutRedirectUri = postLogoutRedirectUri(request, clientRegistration);
				targetUrl = endpointUri(endSessionEndpoint, idToken, postLogoutRedirectUri);
			}
return (targetUrl != null) ? targetUrl : super.determineTargetUrl(request, response);
```
endSessionEndpoint 이 url 로 호출을 하게 되면 인가서버에서도 세션을 삭제하게 됩니다 


```
http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/logout

```
이 값이 openid 로그아웃을 위한 주소가 되는것이고 


```
String idToken = idToken(authentication);
String postLogoutRedirectUri = postLogoutRedirectUri(request, clientRegistration);
targetUrl = endpointUri(endSessionEndpoint, idToken, postLogoutRedirectUri);
```

인증객체에서 idtoken 을 뽑아오고 세션제거가 끝났을시 되돌아올 redirect url 을 적어주고 요청을 만들게 되면 
이 targeturl 는 이렇게 만들어지게 됩니다 get 요청으로 만들고 id_token 과 redirect 주소를 같이 포함시켜서 보내게 되는데 


## AbstractAuthenticationTargetUrlRequestHandler
```
protected void handle(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
			throws IOException, ServletException {
	String targetUrl = determineTargetUrl(request, response, authentication);
	if (response.isCommitted()) {
		this.logger.debug(LogMessage.format("Did not redirect to %s since response already committed.", targetUrl));
		return;
	}
	this.redirectStrategy.sendRedirect(request, response, targetUrl);
}
```

이곳으로 들어와 요청정보를 전송하게 됩니다 그리고 돌아오게 되면 

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/dbbd2013-f4ff-4bc4-b67a-d7ae76d8a091)

이렇게 로그인 화면으로 돌아오게 되고 

![7](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/b4800ed6-b784-44d1-80f2-c65742193cd8)

인가서버에 session 오 지워지게 되고 

![8](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/30e6c3bc-2860-417c-a0d8-ca7460cd4c9a)

그리고 다시 로그인을 할려고 클릭을 하게 되면 원래라면 인가서버에 세션정보가 남아있어서 따로 로그인 절차가 필요없어졌지만 이제는 세션정보가 아예 없어졌기 때문에 
(모든 기기에서 로그아웃) 다시 처음부터 인증을 해야 하는 상태로 돌아오게 됩니다 

여기 까지 OIDC 로그아웃에 대해서 알아보았습니다