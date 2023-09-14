---

title: Spring Secuirty 13 나만의 로그인 및 회원가입 만들기 3
author: kimdongy1000
date: 2023-06-06 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

소스 주소 : https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_0903?ref_type=heads

작업 주소 : https://gitlab.com/kimdongy1000/spring_security_web/-/commit/29c0fdd6b94a0c246b7228e38a478af96d38e4c0


지난시간에 우리는 csrf 에 대해서 알아보았고 우리는 다시 돌아왔가 이제는 회원가입 로직을 만들었니 이제 이 내용을 바탕으로 로그인 하는 방식에 대해서 알아볼려고 합니다 

먼저 화면과 html 을 먼저 만들어보면 

## login.html 


```

<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <link href="/bootStrap/css/bootstrap.min.css" rel="stylesheet">

    <link href="/resources/css/login.css" rel="stylesheet">

    <title>로그인 페이지</title>
</head>
<body>


<main class="form-signin">
    <form id = "login_form" method="post" action="/login/">
        <h1 class="h3 mb-3 fw-normal"> 로그인 </h1>

        <div class="form-floating">
            <input type="email" class="form-control" id="floatingInput" placeholder="name@example.com" name = "email">
            <label for="floatingInput">Email address</label>
        </div>
        <div class="form-floating">
            <input type="password" class="form-control" id="floatingPassword" placeholder="Password" name = "password">
            <label for="floatingPassword">Password</label>
        </div>

        <input  th:name="${_csrf.parameterName}" th:value="${_csrf.token}">


        <button id = "btn_login" class="w-100 btn btn-lg btn-primary" type="button"> 로그인 </button>
    </form>
</main>






<script src="/bootStrap/js/bootstrap.bundle.min.js"></script>
<script src="/resources/js/login.js"></script>
</body>
</html>



```

기존에 있는 signUp 페이지에서 필요한것들만 가져왔습니다 

## login.css
```

html,
body {
  height: 100%;
}

body {
  display: flex;
  align-items: center;
  padding-top: 40px;
  padding-bottom: 40px;
  background-color: #f5f5f5;
}

.form-signin {
  width: 100%;
  max-width: 330px;
  padding: 15px;
  margin: auto;
}

.form-signin .checkbox {
  font-weight: 400;
}

.form-signin .form-floating:focus-within {
  z-index: 2;
}

.form-signin input[type="email"] {
  margin-bottom: -1px;
  border-bottom-right-radius: 0;
  border-bottom-left-radius: 0;
}

.form-signin input[type="password"] {
  border-top-left-radius: 0;
  border-top-right-radius: 0;
}

.form-signin input[type="text"] {
  margin-top: 20px;
  border-top-left-radius: 0;
  border-top-right-radius: 0;
}


```

마찬가지입니다 기존 register.css 에서 그냥 그대로 복사 붙여 넣기 했습니다 

## securityConfig 

```

@Bean
public SecurityFilterChain  securityFilterChain(HttpSecurity httpSecurity) throws Exception{

	httpSecurity.authorizeRequests().antMatchers("/" , "/signUp/*" , "/bootStrap/css/**" , "/bootStrap/js/**" , "/resources/css/**","/resources/js/**").permitAll()
					.anyRequest().authenticated();

	httpSecurity.formLogin()
			.loginPage("/login/")
			.loginProcessingUrl("/login/")
			.usernameParameter("email")
			.passwordParameter("password")
			.permitAll();

	return httpSecurity.build();
}

```

이제 httpSecurity.formLogin() 에서 체인 패턴으로 더 붙었습니다 

loginPage 는 어떤 로그인 페이지를 쓸것인지에 대한 핸들러 규정입니다 
loginProcessingUrl 로그인이 시작되는 url 주소입니다 그럼 loginPage 랑 햇갈릴 수 있는데 loginPage GET 방식으로 loginProcessingUrl post 방식으로 움직입니다 
usernameParameter UsernamePasswordAuthenticationFilter 에서 username 파라미터를 어떤 쿼리 스트링으로 들어오는지 지정합니다 저는 email 로 보낼것입니다 
passwordParameter UsernamePasswordAuthenticationFilter 에서 password 파라미터를 어떤 쿼리 스트링으로 들어오는지 지정하고 저는 password 입니다 

그리고 이때 로그인 인증된 곳은 전부 permitAll 로 지정을 합니다 

## LoginController.java

```
package com.cybb.main.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("login")
public class LoginController {

    @GetMapping("/")
    public String loginPage(){

        return "login";
    }
}


```

이 부부은 아까 우리가 loginPage 매핑되는 핸들러를 지정해줍니다 그럼 시큐리티는 이 url 를 호출해서 로그인 페이지를 띄어주게 됩니다 그럼 화면은 

![로그인 페이지](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/e595fb05-a64c-413e-b773-00bcebce164d)

이렇게 나오게 됩니다 그럼 여기서 끝은 아닙니다 우리가 이제 AuthenProvier 를 커스텀을 해주어야 합니다 

## CustomAuthenticationManger.java

```

package com.cybb.main.User;

import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;

public class CustomAuthenticationManger implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        return null;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return false;
    }
}


```

우리는 CustomAuthenticationManger 에 AuthenticationProvider 를 구현하는 구현체를 만들것입니다 이는 우리가 앞에서 보았다 

`public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean `

이렇게 생긴것을 그리고 Filter 에서 넘어오는 데이터를 가지고 인증을 하게 됩니다 물론 ProviderManager 가 시큐리티에서 제공하는 정석이지만 얼마든지 커스텀을 할 수 있습니다 
그중에서 우리는 인증과 비밀번호를 지우는 정도만 하겠습니다 

그러면 

```

package com.cybb.main.User;

import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.stereotype.Component;

@Component
public class CustomAuthenticationManger implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        System.out.println(authentication);



        return null;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}


```

채워놓고 authentication 봐야 합니다 이떄 중요한것은 

## SecurityFilterChain 에 Provider 명시 
```

@Bean
public SecurityFilterChain  securityFilterChain(HttpSecurity httpSecurity) throws Exception{

	httpSecurity.authorizeRequests().antMatchers("/" , "/signUp/*" , "/bootStrap/css/**" , "/bootStrap/js/**" , "/resources/css/**","/resources/js/**").permitAll()
					.anyRequest().authenticated();

	httpSecurity.formLogin()
			.loginPage("/login/")
			.loginProcessingUrl("/login/")
			.usernameParameter("email")
			.passwordParameter("password")
			.permitAll();


	httpSecurity.authenticationProvider(customAuthenticationManger);

	return httpSecurity.build();
}

```

`httpSecurity.authenticationProvider(customAuthenticationManger);` 이렇게 함으로서 우리는 이제 커스텀한 Provider 를 사용할 준비가 되어 있는것입니다 

우리는 일단 디버깅을 UsernamePasswordAuthenticationFilter 에 하나 걸것이고 ProviderManager 에 하나 걸것입니다 

그럼 안에 내용 상관없이 버튼을 누르면 바로 로그인 호출이 하게끔 만들어서 진행을 하겠습니다 


## UsernamePasswordAuthenticationFilter 

```

@Override
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

이곳이 제일 먼저 호출이 됩니다 그럴 수 밖에 없는것은 일단 Filter 는 등록만 해두면 mvc 요청이 왔을때 왠만하면 한번은 다 거치기 때문에 이곳에 안들어 올 순 없습니다 
그러면 이 내용들은 다 알지만 한번씩 더 살펴보면 

if (this.postOnly && !request.getMethod().equals("POST")) 로그인은 반드시 post 요청으로 들어와야 하고 

이곳에서 요청정보의 username , password 를 꺼내게 됩니다 그리고 unauthenticated 를 호출해서 미인증토큰만 만들고 이 토큰을 providerManger 에게 요청을 하게 됩니다 


## ProviderManager
```

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

이곳으로 들어오게 되는데 이곳에서 재미있는 일이 생기게 됩니다 

int size = this.providers.size(); 이 부분에 오게 되면 인증을 처리하기 위해서 provider 가 제공이 되는데 이게 몇개 있는지 살펴보게 됩니다 

```

providers = {ArrayList@13116}  size = 2
 0 = {CustomAuthenticationManger@13138} 
  Class has no fields
 1 = {AnonymousAuthenticationProvider@13230} 
  messages = {MessageSourceAccessor@13231} 
  key = "1e9c230b-cee7-4441-9a3d-11dcbc7f46e5"

```

이 안에 두개를 살펴보면 아직 아무런 내용이 없는 CustomAuthenticationManger와 AnonymousAuthenticationProvider 이 존재하게 됩니다 
AnonymousAuthenticationProvider 같은 경우는 익명사용자 즉 인증이 되지 않은 익명사용자를 인증해서 사용합니다 이에 대해서는 다음에 다룰 기회가 있을예정입니다 

그리고 우리가 앞에서 httpSecurity.authenticationProvider(customAuthenticationManger); 주었기 때문에 기존에 처리하는 권한부여 는 사용되지 않게 됩니다 
만약 우리가 커스텀해서 사용하지 않으면 이 부분에서는 DaoAuthenticationProvider 사용되게 됩니다 


```

providers = {ArrayList@12612}  size = 1
 0 = {DaoAuthenticationProvider@12631} 
  passwordEncoder = {BCryptPasswordEncoder@12632} 
  userNotFoundEncodedPassword = null
  userDetailsService = {InMemoryUserDetailsManager@12633} 
  userDetailsPasswordService = {InMemoryUserDetailsManager@12633} 
  logger = {LogAdapter$Slf4jLocationAwareLog@12634} 
  messages = {MessageSourceAccessor@12635} 
  userCache = {NullUserCache@12636} 
  forcePrincipalAsString = false
  hideUserNotFoundExceptions = true
  preAuthenticationChecks = {AbstractUserDetailsAuthenticationProvider$DefaultPreAuthenticationChecks@12637} 
  postAuthenticationChecks = {AbstractUserDetailsAuthenticationProvider$DefaultPostAuthenticationChecks@12638} 
  authoritiesMapper = {NullAuthoritiesMapper@12639} 

```

이 인증부분에 대한것은 우리가 지난시간에 한번 다루어 보았으니 참고 바랍니다 

## CustomAuthenticationManger 마저 작성 

```

package com.cybb.main.User;

import com.cybb.main.entity.UserEntity;
import com.cybb.main.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.authority.mapping.SimpleAuthorityMapper;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;


@Component
public class CustomAuthenticationManger implements AuthenticationProvider {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;


    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        String username = authentication.getName();

        if(!StringUtils.hasText(username)){
            throw new UsernameNotFoundException("username 이 존재하지 않습니다");
        }

        String password = authentication.getCredentials().toString();

        if(!StringUtils.hasText(password)){
            throw new UsernameNotFoundException("password 가 존재하지 않습니다");
        }

        Optional<UserEntity> opsUserEntity  = userRepository.findByUsername(username);



        if(!opsUserEntity.isPresent()){
            throw new RuntimeException("존재하지 않는 회원입니다");
        }

        UserEntity userEntity = opsUserEntity.get();

        if(!passwordEncoder.matches(password , userEntity.getPassword())){
            throw new RuntimeException("비밀번호가 서로 다릅니다.");
        }

        List<GrantedAuthority> ADMIN_AUTHORITIES = Arrays.asList(new SimpleGrantedAuthority("ADMIN"));

        UserDetails User = new User(userEntity.getUsername() , userEntity.getPassword() ,  ADMIN_AUTHORITIES);

        UsernamePasswordAuthenticationToken authenticationToken = UsernamePasswordAuthenticationToken.authenticated(User , authentication.getCredentials() , ADMIN_AUTHORITIES);


        return authenticationToken;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}


```

우리는 DaoAuthenticationProvider 가 아닌 우리가 직접 만든 CustomAuthenticationManger 를 사용하여 인증을 마무리 짓도록 하겠습니다 
AuthenticationProvider 구현체로 받아서 구현을 하게 되면 public Authentication authenticate(Authentication authentication) throws AuthenticationException 
반드시 구현을 해야겠금 내려오기 때문에 여기에서 우리는 인증을 마무리 해주면됩니다 

우리가 가져올 DB 는 JPA 의 UserEntity 에서 가져오는 것임으로 `Optional<UserEntity> opsUserEntity  = userRepository.findByUsername(username);` 데이터를 조회후 
간단한 로직만 맞추고 마지막에는 

```
UsernamePasswordAuthenticationToken authenticationToken = UsernamePasswordAuthenticationToken.authenticated(User , authentication.getCredentials() , ADMIN_AUTHORITIES);

```
사용해서 토큰객체를 만들어서 return 하면됩니다 그럼 이 토큰객체는 자연스럽게 시큐리티 컨텍스트에 들어가게 됨으로 그 이후 로직은 시큐리티가 알아서 만들어주게 됩니다 
그러면 우리는 인증된 객체를 가지고 로그인해서 demo 를 요청하면 우리가 넣은 데이터 그대로 잘 나오는것을 확인할 수 있습니다 

여기 까지 우리는 우리가 사용하고 싶은 회원가입 페이지 로그인 페이지를 만들고 인증을 커스텀하게 만들어보았습니다 



