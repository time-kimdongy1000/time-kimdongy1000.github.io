---

title: Spring Secuirty 2 
author: kimdongy1000
date: 2023-05-13 10:49
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

우리는 지난시간에 간단한 api 를 만들고 요청을 했을때 인증이 없는 사람에게 시큐리티는 로그인 페이지로 이동을 시켯다 사실 이런 설정은 우리가 하지 않았지만
시큐리티 내부 로직에는 인증 / 인가가 없는 사람에 대한 시큐리티 프레임워크가 어떻게 이 사람을 다시 인증 인가를 받게 하는지 알아볼것이다 

## Filter 
시큐리티는 Filter 의 집합이다 특정 요청사항에 따라서 수많은 Filter 가 얽혀 있는데이 Fiter 가 인증 인가 되지 않은 사람을 어떻게 걸러내는지 오늘부터 알아볼 예정이다 
물론 모든 Filter 를 이 잡듯이 잡아서 검증 하지는 않을 예정입니다 

## FilterSecurityInterceptor 

첫번쨰 상황은 미인증된 유저가 /demo 요청을 넣었을때 Filter 가 어떻게 사용자를 로그인페이지로 인도하는지 보겠습니다 

```
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter 

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		invoke(new FilterInvocation(request, response, chain));
	}


```

Filter 를 구현하고 있기 때문에 무조건 이 doFilter 에 요청이 들어가게 됩니다 그럼이 invoke 함수를 호출하는 순간

```
public class ExceptionTranslationFilter extends GenericFilterBean implements MessageSourceAware 

private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		try {
			chain.doFilter(request, response);
		}
		catch (IOException ex) {
			throw ex;
		}
		catch (Exception ex) {
			// Try to extract a SpringSecurityException from the stacktrace
			Throwable[] causeChain = this.throwableAnalyzer.determineCauseChain(ex);
			RuntimeException securityException = (AuthenticationException) this.throwableAnalyzer
					.getFirstThrowableOfType(AuthenticationException.class, causeChain);
			if (securityException == null) {
				securityException = (AccessDeniedException) this.throwableAnalyzer
						.getFirstThrowableOfType(AccessDeniedException.class, causeChain);
			}
			if (securityException == null) {
				rethrow(ex);
			}
			if (response.isCommitted()) {
				throw new ServletException("Unable to handle the Spring Security Exception "
						+ "because the response is already committed.", ex);
			}
			handleSpringSecurityException(request, response, chain, securityException);
		}
	}



```

이 쪽으로 오게 됩니다 이떄 평범한 try 문으로 들어가는것이 아니라 catch 문장으로 빠져나가게 되는데요 이떄 에러명은 
org.springframework.security.access.AccessDeniedException: Access is denied 즉 권한이 거절되었다는 뜻입니다 


```

private void handleAccessDeniedException(HttpServletRequest request, HttpServletResponse response,
			FilterChain chain, AccessDeniedException exception) throws ServletException, IOException {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		boolean isAnonymous = this.authenticationTrustResolver.isAnonymous(authentication);
		if (isAnonymous || this.authenticationTrustResolver.isRememberMe(authentication)) {
			if (logger.isTraceEnabled()) {
				logger.trace(LogMessage.format("Sending %s to authentication entry point since access is denied",
						authentication), exception);
			}
			sendStartAuthentication(request, response, chain,
					new InsufficientAuthenticationException(
							this.messages.getMessage("ExceptionTranslationFilter.insufficientAuthentication",
									"Full authentication is required to access this resource")));
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace(
						LogMessage.format("Sending %s to access denied handler since access is denied", authentication),
						exception);
			}
			this.accessDeniedHandler.handle(request, response, exception);
		}
	}


```

그리고 이 메서드는 handleAccessDeniedException 요청에 따라 넘어오게 되며 sendStartAuthentication() 함수를 호출하게 됩니다 이름에서 알 수 있다 싶히 
인가를 받기 위한 핸들러로 보이며 

```

protected void sendStartAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
			AuthenticationException reason) throws ServletException, IOException {
		// SEC-112: Clear the SecurityContextHolder's Authentication, as the
		// existing Authentication is no longer considered valid
		SecurityContext context = SecurityContextHolder.createEmptyContext();
		SecurityContextHolder.setContext(context);
		this.requestCache.saveRequest(request, response);
		this.authenticationEntryPoint.commence(request, response, reason);
	}

```

this.requestCache.saveRequest(request, response); 현재 요청 (/demo) 를 캐싱해서 저장을 해두고 
this.authenticationEntryPoint.commence(request, response, reason); 호출하게 됩니다 


그런데 AuthenticationEntryPoint 는 인터페이스이기 떄문에 구현체 LoginUrlAuthenticationEntryPoint 으로 넘어오게 되는데 

```
public class LoginUrlAuthenticationEntryPoint implements AuthenticationEntryPoint, InitializingBean {

```

```

if (!this.useForward) {
	// redirect to login page. Use https if forceHttps true
	String redirectUrl = buildRedirectUrlToLoginPage(request, response, authException);
	this.redirectStrategy.sendRedirect(request, response, redirectUrl);
	return;
}

```

이 문장을 통해서 redirectUrl=http://localhost:8080/login 이 찍히게 됩니다 그렇기에 인증 / 인가가 없는 요청에 대해서는 시큐리티 프레임워크가 다시 인가  인증을 받게 하기 위해 
로그인 페이지로 redirect 하는 모습을 볼 수 있습니다 

우리는 정말 간단하게 인가 인증이 없는 계정으로 요청을 넣었을때 어떤식으로 해서 로그인 화면으로 되돌아오게 되는지 알게 되었습니다.

다음시간에는 어떻게 해서 우리가 만들지 않은 로그인 form 생성에 대해서 알아보겠습니다 