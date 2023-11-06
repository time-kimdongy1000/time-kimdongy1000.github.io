---
title: Spring Secuirty 3 기본 로그인 페이지 
author: kimdongy1000
date: 2023-05-15 10:49
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

우리는 지난시간에 권한이 없는 유저가 권한이 필요한 api 를 호출할때 어떤 식으로 로그인 페이지로 인도하는지 보았다 다만 여기서 의문점은 우리는 로그인 페이지를 만들지 않았는데 시큐리티는 알아서 로그인 핸들러와 , 로그인 페이지를 인도하는 모습을 보여주었다 어떻게 이런일이 가능할까?

![스크린샷 2023-08-06 105416](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/39942fb0-5695-4970-ab8e-50a4607773fb)

우리가 최초로 로그인을 할려고 하면 이런 화면이 보이게 됩니다 이런 화면은 DefaultLoginPageGeneratingFilter 로 생성이 되게 되는데 

## DefaultLoginPageGeneratingFilter

```
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		boolean loginError = isErrorPage(request);
		boolean logoutSuccess = isLogoutSuccess(request);
		if (isLoginUrlRequest(request) || loginError || logoutSuccess) {
			String loginPageHtml = generateLoginPageHtml(request, loginError, logoutSuccess);
			response.setContentType("text/html;charset=UTF-8");
			response.setContentLength(loginPageHtml.getBytes(StandardCharsets.UTF_8).length);
			response.getWriter().write(loginPageHtml);
			return;
		}
		chain.doFilter(request, response);
	}
```

이 클래스에는 이와 같은 핸들러가 있습니다 

여기 안에 있는 소스중 String loginPageHtml = generateLoginPageHtml(request, loginError, logoutSuccess); 이런 형식이 있는데 이는 generateLoginPageHtml 매소드 명에서 알 수 있다 싶히 로그인 페이지를 생성한다는 뜻으로 해석이 되는데 

```
private String generateLoginPageHtml(HttpServletRequest request, boolean loginError, boolean logoutSuccess) {

		String errorMsg = "Invalid credentials";
		if (loginError) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				AuthenticationException ex = (AuthenticationException) session
						.getAttribute(WebAttributes.AUTHENTICATION_EXCEPTION);
				errorMsg = (ex != null) ? ex.getMessage() : "Invalid credentials";
			}
		}
		String contextPath = request.getContextPath();
		StringBuilder sb = new StringBuilder();
		sb.append("<!DOCTYPE html>\n");

		...

		if (this.formLoginEnabled) {
		sb.append("      <form class=\"form-signin\" method=\"post\" action=\"" + contextPath
				+ this.authenticationUrl + "\">\n");
		sb.append("        <h2 class=\"form-signin-heading\">Please sign in</h2>\n");
		sb.append(createError(loginError, errorMsg) + createLogoutSuccess(logoutSuccess) + "        <p>\n");
		sb.append("          <label for=\"username\" class=\"sr-only\">Username</label>\n");
		sb.append("          <input type=\"text\" id=\"username\" name=\"" + this.usernameParameter
				+ "\" class=\"form-control\" placeholder=\"Username\" required autofocus>\n");
		sb.append("        </p>\n");
		sb.append("        <p>\n");
		sb.append("          <label for=\"password\" class=\"sr-only\">Password</label>\n");
		sb.append("          <input type=\"password\" id=\"password\" name=\"" + this.passwordParameter
				+ "\" class=\"form-control\" placeholder=\"Password\" required>\n");
		sb.append("        </p>\n");
		sb.append(createRememberMe(this.rememberMeParameter) + renderHiddenInputs(request));
		sb.append("        <button class=\"btn btn-lg btn-primary btn-block\" type=\"submit\">Sign in</button>\n");
		sb.append("      </form>\n");
}
		...

		sb.append("</body></html>");
		return sb.toString();
	}

```

`if (this.formLoginEnabled)` 이 if문을 통해서 우리가 현재 하고 있는 from 로그인을 사용하고 있기에 이 부분이 true 가 되어서 렌러링이 되게 됩니다 
다른것을도 렌더링할 수 있지만 지금은 사용하지 않기에 전부 false 가 되어서 렌더링을 하지 않습니다 

formLoginEnabled 로 인해서 지금처럼 로그인 form 을 만들게 되는것이죠 그리고 renderHiddenInputs 이라는 메서드를 통해서 필요한 몇가지 태그를 더 넣게 되는데 
이떄 제일 대표적으로 들어가는 input 이 csrf 필터입니다 이 csrf Filter 는 다음에 설명할 일이 있을예정이고 

```
<input name="_csrf" type="hidden" value="7ade26eb-23dc-4d32-adaa-3165e3685629">

```

이런 결과를 거쳐서 우리가 지금 보고 있는 로그인 페이지를 보게 되는것입니다.