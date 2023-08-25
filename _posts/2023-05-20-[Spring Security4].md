---

title: Spring Secuirty 4 로그아웃 , 로그아웃 페이지 
author: kimdongy1000
date: 2023-05-20 10:49
categories: [Spring, Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

우리는 지난시간까지 간단한 로그인 및 로그인 페이지가 생기게 된 배경 그리고 임시로 발급된 계정으로 로그인을 해보았습니다 이번시간에는 로그아웃을 진행을 하겠습니다

![로그아웃](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/090c7be4-d31a-453b-a2ff-479d614a3edf)

기본적인 로그아웃 Handler 는 localhost:8080/logout 로 지정하며 이때 위와 같은 페이지가 뜨며 로그아웃을 진행하게 됩니다 

![로그아웃2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/9789219a-f488-4dbc-b288-8130747c2b01)


이렇게 변하게 되는데 오늘은 이페이지가 어떻게 만들어지고 또 로그아웃 Filter 는 어떻게 동작하는지에 대해서 알아보겠습니다 

## DefaultLogoutPageGeneratingFilter

```
public class DefaultLogoutPageGeneratingFilter extends OncePerRequestFilter 

```
기본적으로 로그아웃페이지는 DefaultLogoutPageGeneratingFilter 이와같은 Filter 에서 생겨나게 된다 

```

protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		if (this.matcher.matches(request)) {
			renderLogout(request, response);
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace(LogMessage.format("Did not render default logout page since request did not match [%s]",
						this.matcher));
			}
			filterChain.doFilter(request, response);
		}
	}

```
이때 doFilterInternal 로 renderLogout 을 호출하게 되는데 

```

private void renderLogout(HttpServletRequest request, HttpServletResponse response) throws IOException {
		StringBuilder sb = new StringBuilder();
		sb.append("<!DOCTYPE html>\n");
		sb.append("<html lang=\"en\">\n");
		sb.append("  <head>\n");
		sb.append("    <meta charset=\"utf-8\">\n");
		sb.append("    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1, shrink-to-fit=no\">\n");
		sb.append("    <meta name=\"description\" content=\"\">\n");
		sb.append("    <meta name=\"author\" content=\"\">\n");
		sb.append("    <title>Confirm Log Out?</title>\n");
		sb.append("    <link href=\"https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css\" "
				+ "rel=\"stylesheet\" integrity=\"sha384-/Y6pD6FV/Vv2HJnA6t+vslU6fwYXjCFtcEpHbNJ0lyAFsXTsjBbfaDjzALeQsN6M\" "
				+ "crossorigin=\"anonymous\">\n");
		sb.append("    <link href=\"https://getbootstrap.com/docs/4.0/examples/signin/signin.css\" "
				+ "rel=\"stylesheet\" crossorigin=\"anonymous\"/>\n");
		sb.append("  </head>\n");
		sb.append("  <body>\n");
		sb.append("     <div class=\"container\">\n");
		sb.append("      <form class=\"form-signin\" method=\"post\" action=\"" + request.getContextPath()
				+ "/logout\">\n");
		sb.append("        <h2 class=\"form-signin-heading\">Are you sure you want to log out?</h2>\n");
		sb.append(renderHiddenInputs(request)
				+ "        <button class=\"btn btn-lg btn-primary btn-block\" type=\"submit\">Log Out</button>\n");
		sb.append("      </form>\n");
		sb.append("    </div>\n");
		sb.append("  </body>\n");
		sb.append("</html>");
		response.setContentType("text/html;charset=UTF-8");
		response.getWriter().write(sb.toString());
	}

```
이 renderLogout 은 이와 같은 방식으로 제작이 되며 logout 으로 요청이 들어갈떄 다만 이때 중요한것은 로그인 , 로그아웃을 내가 커스텀하게 만들 수 있기 때문에 
요청정보가 여기 걸리지 않으면 커스텀한 로그아웃 페이지를 제공하게 된다 그에 대한 설명은 잠시 미뤄두고 오늘은 기본적으로 제공한 로그아웃 페이지만 살펴볼 예정입니다 

```
if (this.matcher.matches(request))

matcher = {AntPathRequestMatcher@6164} "Ant [pattern='/logout', GET]"
 matcher = {AntPathRequestMatcher$SpringAntMatcher@6175} 
 pattern = "/logout"
 httpMethod = {HttpMethod@6177} "GET"
 caseSensitive = true
 urlPathHelper = null

```
결과를 보면 이와 같이 /logout GET 이렇게 표현이 되고 있습니다 그리고 하단 renderLogout 호출하고 있고 이를 던지면서 그 페이지로 인도하는 모습을 보여주고 있습니다 
그럼 보시는것과 같이 로그아웃 페이지가 나오게 되고 우리는 상단에 Log out 버튼을 클릭하게 되면 

## LogoutFilter 

```
public class LogoutFilter extends GenericFilterBean


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

```
으로 호출이 들어오게 되는데 

```

Authentication auth = SecurityContextHolder.getContext().getAuthentication();


```
이때 시큐리티에서 중요한 객체가 한가지 나오게 됩니다 Authentication 이는 시큐리티에서 인증/인가 와 관련된 객체이며 인증된 객체는 여기에 저장이 되게 됩니다 
이에 대해서도 역시 자세하게 다룰 날이 올것이며 오늘은 이런 객체가 있다만 알고 계시면됩니다 

```
this.handler.logout(request, response, auth);
```
이때 이 부분이 로그아웃을 담당하게 되는데 로그아웃도 단순 로그아웃 뿐만 아니라 여러가지 과정을 진행을 하게 되는데 
세션제거 , 쿠키제거, CSRF 제거 등다양한 과정을 거치는데 그중에서 저는 세션제거만 보도록 하겠습니다 

```

public class SecurityContextLogoutHandler implements LogoutHandler 


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


```
이는 현제 또 중요한 객체 SecurityContext 에서 인증된 사용자를 제거하는 것을 담당하는 클래스입니다 

this.invalidateHttpSession 세션이 초기화 되어 있지 않으면 session.invalidate(); 세션을 무효화 시키고 

```
SecurityContext context = SecurityContextHolder.getContext();
SecurityContextHolder.clearContext();

```
이 부분이 시큐리티에서 인증을 담당하는 객체부분을 말끔지 지우는 역활을 합니다 
context.setAuthentication(null); 이부분도 컨텍스트 안에 있는 인증부분을 말끔히 지우는 역활을 하는것이죠


```

public class DelegatingLogoutSuccessHandler implements LogoutSuccessHandler

@Override
public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
		throws IOException, ServletException {
	for (Map.Entry<RequestMatcher, LogoutSuccessHandler> entry : this.matcherToHandler.entrySet()) {
		RequestMatcher matcher = entry.getKey();
		if (matcher.matches(request)) {
			LogoutSuccessHandler handler = entry.getValue();
			handler.onLogoutSuccess(request, response, authentication);
			return;
		}
	}
	if (this.defaultLogoutSuccessHandler != null) {
		this.defaultLogoutSuccessHandler.onLogoutSuccess(request, response, authentication);
	}
}

```

이떄 `if (this.defaultLogoutSuccessHandler != null)` 부분을 타게 되면서 this.defaultLogoutSuccessHandler.onLogoutSuccess(request, response, authentication);

이 부분을 호출하게 되는데 이 부분을 따라가면

```

public abstract class AbstractAuthenticationTargetUrlRequestHandler 


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

이 부분에 String targetUrl = determineTargetUrl(request, response, authentication); /login?logout 값이 나오게 됩니다 즉 다시 /login 페이지로 핸들러를 동작을 시키고 
이전처럼 DefaultLoginPageGeneratingFilter 를 통해서 다시 새로운 로그인 페이지를 생성게되고 페이지를 return 해서 

결국 로그아웃 후 로그인은 위와같은 페이지로 이동을 하게 됩니다 

그럼 우리는 앞전것을 포함해서 간단한 로그인 / 로그아웃을 경험하게 되었습니다 







