---
title: Spring Secuirty 2 
author: kimdongy1000
date: 2023-05-13 10:49
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

우리는 지난시간에 간단한 api 를 만들고 요청을 했을때 인증이 없는 사람에게 시큐리티는 로그인 페이지로 리다이렉트 된 상황을 보았습니다 이런 설정은 우리가 하지 않았지만
시큐리티 내부 로직에는 인증 / 인가가 없는 사람에 대한 로직 처리를 어떻게 처리할지 설정이 다 되어 있는 상태입니다 시큐리티 프레임워크가 이 페이지에서는 인증/인가가 없는 유저에 대해서 어떻게 리다이렉트 시키는지에 대해서 알아보겠습니다 

## Filter 

시큐리티는 Filter 의 집합이다 특정 요청사항에 따라서 수 많은 Filter 가 얽혀 있는데 이 Filter 를 통가하지 못하게 되면 시큐리티는 인증/인가 허락이 없었기 때문에 다시금 로그인을 유도하게 됩니다 


## FilterSecurityInterceptor 
```
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter 

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		invoke(new FilterInvocation(request, response, chain));
	}

```
Filter 를 상속또는 구현을 받게 되면 모든 요청은 Filter 를 무조건 거치게 되어 있습니다 

## ExceptionTranslationFilter
```
public class ExceptionTranslationFilter extends GenericFilterBean implements MessageSourceAware {

private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		try {
			chain.doFilter(request, response);
		}
		catch (IOException ex) {
			throw ex;
		}
		catch (Exception ex) {
			
			...

			handleSpringSecurityException(request, response, chain, securityException);
		}
	}
}	

```
ExceptionTranslationFilter 는 시큐리티에서 예외를 처리하고, 예외에 대한 적절한 응답을 생성하는 역할을 하는 필터입니다 이 Filter 를 거치면 보통 2가지의 Exception 을 처리하게 되는데 
이때 처리되는 핸들러는 하단 메서드 handleAccessDeniedException 에서 처리를 진행을 하게 됩니다 이때 처리되는것은 대부분 2개의 Exception 으로 처리가 되는데 

1. handleAuthenticationException
	이는 인증에 실패한 경우이 예외처리를 시도하며 이때는 응답코드 401 (Unauthorized) 로 반환하게 됩니다 

2. handleAccessDeniedException
	이는 인증에는 성공했지만 해당 페이지 방문을 거절하는 것입니다 이때는 응답코드 403 (Forbidden) 으로 반환하게 됩니다 


## handleSpringSecurityException 
```
private void handleAccessDeniedException(HttpServletRequest request, HttpServletResponse response,
			FilterChain chain, AccessDeniedException exception) throws ServletException, IOException {
		
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		boolean isAnonymous = this.authenticationTrustResolver.isAnonymous(authentication);

		if (isAnonymous || this.authenticationTrustResolver.isRememberMe(authentication)) {
			...
			sendStartAuthentication(request, response, chain,
					new InsufficientAuthenticationException(
							this.messages.getMessage("ExceptionTranslationFilter.insufficientAuthentication",
									"Full authentication is required to access this resource")));
		}
		else {
			...
			this.accessDeniedHandler.handle(request, response, exception);
		}
	}


```
handleAccessDeniedException 는 로그인 페이지로 Redirect 하지 않습니다 그 이유는 이 사람은 인증은 받았지만 해당 페이지 열람할 권한이 없기 때문이기 때문에 다시 인증을 받지는 않지만 
handleAuthenticationException 은 애초에 인증을 받은 사람이 아니기 때문에 스프링은 이 사람에 대해서 다시 한번 인증을 받게끔 `sendStartAuthentication` 호출 하여 로그인 페이지로 이동을 시키게 됩니다 이때 우리가 앞에서 보았던 아무런 인증없이 바로 `localhost:8080/demo` 를 요청했을때 시큐리티는 이런 내부적인 로직에 따라서 사용자를 로그인 페이지로 인도를 하게 됩니다 


```
protected void sendStartAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
			AuthenticationException reason) throws ServletException, IOException {

		SecurityContext context = SecurityContextHolder.createEmptyContext();
		SecurityContextHolder.setContext(context);
		this.requestCache.saveRequest(request, response);
		this.authenticationEntryPoint.commence(request, response, reason);
}
```

this.requestCache.saveRequest(request, response); 현재 요청 (/demo) 를 캐싱해서 저장을 해두고 
this.authenticationEntryPoint.commence(request, response, reason); 호출하게 됩니다 
그런데 AuthenticationEntryPoint 는 인터페이스이기 떄문에 구현체 LoginUrlAuthenticationEntryPoint 으로 넘어오게 되됩니다

## LoginUrlAuthenticationEntryPoint
```
public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
	if (!this.useForward) {
		String redirectUrl = buildRedirectUrlToLoginPage(request, response, authException);
		this.redirectStrategy.sendRedirect(request, response, redirectUrl);
		return;
	}
}

```

이 문장을 통해서 redirectUrl=http://localhost:8080/login 이 찍히게 됩니다 그렇기에 인증 / 인가가 없는 요청에 대해서는 시큐리티 프레임워크가 다시 인가  인증을 받게 하기 위해 로그인 페이지로 redirect 하는 모습을 볼 수 있습니다 

우리는 이렇게 시큐티티에서 인증 인가를 받지 못한 유저가 어떻게 다시 로그인 페이지로 나가게 되는지에 대해서 알아보았습니다