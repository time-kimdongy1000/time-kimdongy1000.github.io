---

title: Spring Secuirty 12 CsrfFilter
author: kimdongy1000
date: 2023-06-05 16:00
categories: [Spring, Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

지난시간 우리는 csrfFilter 에 대해서 약간 알아보았다 다시 정리해보면 Crsf 공격은 

CSRF(Cross-Site Request Forgery) 토큰은 웹 애플리케이션의 보안을 강화하는 데 사용되는 토큰종류이다 그럼 csrf 공격은 웹사이트 취약점을 이용한 공격의 한가지 방법입니다 

1) 선량한 사용자가 로그인을 하여 정당한 권한을 부여 받습니다 

2) 선량한 사용자는 악의적 사용자가 심어둔 공격페이지를 자신도 모르게 열게됩니다 (게시글 , 메일 등등 )

3) 공격자는 사용자가 공격페이지를 열람해서 얻은 쿠키 또는 세션을 얻게 됩니다 

4) 공격자는 이러한 세션 또는 쿠키를 활용해서 사이트를 공격합니다 이때 서버는 이러한 요청을 선량한 사용자가 요청을 얻은것이라고 판단해서 이를 실행하게 됩니다 

5) 선량한 사용자는 자신도 모르게 공격을 가담하게 된 가담자가 됩니다 

이렇게 CSRF 공격인데 방어하는 로직은 의외로 간단했다 

```
<input  th:name="${_csrf.parameterName}" th:value="${_csrf.token}">

```

하단에 이런 표기를 넣어주고 서버와의 통신때 이 정보를 같이 넘겨주면 되는것이다 그럼 이 csrf 는 어떤식으로 동작하는지에 대해서 알아볼 예정이다 

## CsrfFilter

```
public final class CsrfFilter extends OncePerRequestFilter 


@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		request.setAttribute(HttpServletResponse.class.getName(), response);
		CsrfToken csrfToken = this.tokenRepository.loadToken(request);
		boolean missingToken = (csrfToken == null);
		if (missingToken) {
			csrfToken = this.tokenRepository.generateToken(request);
			this.tokenRepository.saveToken(csrfToken, request, response);
		}
		request.setAttribute(CsrfToken.class.getName(), csrfToken);
		request.setAttribute(csrfToken.getParameterName(), csrfToken);
		if (!this.requireCsrfProtectionMatcher.matches(request)) {
			if (this.logger.isTraceEnabled()) {
				this.logger.trace("Did not protect against CSRF since request did not match "
						+ this.requireCsrfProtectionMatcher);
			}
			filterChain.doFilter(request, response);
			return;
		}
		String actualToken = request.getHeader(csrfToken.getHeaderName());
		if (actualToken == null) {
			actualToken = request.getParameter(csrfToken.getParameterName());
		}
		if (!equalsConstantTime(csrfToken.getToken(), actualToken)) {
			this.logger.debug(
					LogMessage.of(() -> "Invalid CSRF token found for " + UrlUtils.buildFullRequestUrl(request)));
			AccessDeniedException exception = (!missingToken) ? new InvalidCsrfTokenException(csrfToken, actualToken)
					: new MissingCsrfTokenException(actualToken);
			this.accessDeniedHandler.handle(request, response, exception);
			return;
		}
		filterChain.doFilter(request, response);
	}

```

클래스에도 적혀 있다싶이 Filter 를 상속받고 있는 모습이다 CSRF 는 두가지 방식으로 동작하는데 첫번째는 csrf 를 발급받지 않은 페이지에 csrf 를 발급해주고 
두번째는 서버와 같이 csrf 를 token 을 넘겨주면 해당 csrf 유효성을 검증하는 방식 2가지로 이루어진다 

## csrfToken 발급

```
CsrfToken csrfToken = this.tokenRepository.loadToken(request);

boolean missingToken = (csrfToken == null);

if (missingToken) {

    csrfToken = this.tokenRepository.generateToken(request);
    this.tokenRepository.saveToken(csrfToken, request, response);
}
```
최조 load 에서는 요청정보에 crsf token 이 없기 때문에 먼저 토큰을 만드는 작업을 진행하게 됩니다 
`this.tokenRepository.generateToken(request);` 이 요청정보에 맞는 토큰정보를 생성해서 tokenRepository 에 토큰정보와 요청정보 응답정보 모두를 심게 됩니다 

```
request.setAttribute(CsrfToken.class.getName(), csrfToken);
request.setAttribute(csrfToken.getParameterName(), csrfToken);
```

그런다음 요청정보에 csrf 토큰을 심어두고 요청은 끝이나게 됩니다 이때 앞에도 말했다 싶히 서버의 상태를 변경하지 않는 메소드 
POST , DELETE , PATCH , PUT 메서드 호출할때만 동작이 되고 그 외에는 넘기게 됩니다 그 소스는 

```

if (!this.requireCsrfProtectionMatcher.matches(request)) {
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Did not protect against CSRF since request did not match "
                + this.requireCsrfProtectionMatcher);
    }
    filterChain.doFilter(request, response);
    return;
}

```
여기에서 검증을 하게 되고 이곳에거 `"GET", "HEAD", "TRACE", "OPTIONS"` 이런 요청이 올때는 CSRF 검증을 하지 않고 다음 페이지로 넘어가게 됩니다 
그러면 이제 우리가 가는 페이지에 input 태그 안에 csrf 토큰이 생겨나게 됩니다 

## CSRF 토큰검증 
토큰검증은 api 호출과 특정한 메서드 (post , patch , delete , put) 호출이 되면서 헤더 정보에 우리는 csrf 토큰을 심어서 보냈습니다 

```

request.setAttribute(HttpServletResponse.class.getName(), response);
CsrfToken csrfToken = this.tokenRepository.loadToken(request);

```

요청정보에 토큰이 심어저셔 오게 되며 요청정보를 분석해서 csrf 토큰을 분리하게 됩니다 

```

String actualToken = request.getHeader(csrfToken.getHeaderName());

```
여기서 요청정보의 헤더값으로 토큰을 분리해서 가져오게 됩니다 이떄 csrfToken.getHeaderName() 값은 X-CSRF-TOKEN 인 값이고 우리는 앞에서 헤더에 key 값이 X-CSRF-TOKEN 인것으로 토큰을 심어서 보냈습니다 그러면 이제 actualToken 안에는 csrf 토큰이 담기게 됩니다 

```

if (!equalsConstantTime(csrfToken.getToken(), actualToken))

```

토큰 리포지토리에 저장된 토큰과 요청헤더에 담겨있는 토큰을 비교해서 동일한 토큰인지 확인을 하게 됩니다 이때 토큰은 발급된 위치 시간 같은 문자인지를 확인합니다 
실제 발급이 되었다고 할지라도 시간이 넘어가면 그 토큰은 유효하지 않게 됩니다 

토큰이 옳으면 다음페이지 그렇지 않으면 로그인 페이지로 로그아웃 시켜서 인증을 확인하게 합니다 

오늘은 이 csrf 토큰과 filter 에 대해서 공부를 해보았습니다 다음시간에 계속해서 회원가입 및 로그인 로직을 만들어보겠습니다 
