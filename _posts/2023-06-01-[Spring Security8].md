---

title: Spring Secuirty 8 Authorities , Role 
author: kimdongy1000
date: 2023-06-01 10:00
categories: [Spring, Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

지난시간에는 User 를 만들때 사용한 PasswordEcndoer 를 활용했다면 이번시간에는 User 를 만들때 사용하는 Authorities , Role 를 활용한 유저생성 마지막에 대해서 공부를 해보도록 하겠습니다 

## Authorities , Role 의 각각 정의 

```
1) Role 정의 
	이는 주로 이 User 가 가지는 그룹을 나타냅니다 일반 유저를 나타내는 유저 그리고 관리자를 나타내는 Admin 등등 지정할 수 있으며 이는 유저의 컨셉과 사용자의 사용법에 따라서 다르게 정의 됩니다 


2) Authorities 정의 
	이는 주로 권한을 나타냅니다 읽기 권한 쓰기 권한 수정권한 삭제권한 Role 처럼 권한을 나누고 메서드 호출할때 해당 Authorities 를 지정하게 되면 해당 권한을 가지고 있지 않은 유저는 
	해당 메서드를 호출 할 수 없습니다 

```

## User Role , Authorities 정의 

그럼 User를 만들때 어떻게 정의가 되느냐

```

 @Bean
public UserDetailsService createUser(){
	UserDetails user = User.builder()
							.username("user")
							.password(passwordEncoder().encode("1234567890"))
							.authorities("Read")
							.roles("USER").build();

	UserDetails admin_user = User.builder()
							.username("admin")
							.password(passwordEncoder().encode("1234567890"))
							.authorities("Read" , "Write" , "Update" , "Delete" )
							.roles("ADMIN").build();


	return new InMemoryUserDetailsManager(user , admin_user);
}

```

저는 한명의 일반 유저와 다른 한명의 관리자 계정을 만들었습니다 일반 user 는 권한을 Read 밖에 안줄것이고 Admin 권한은 모든 권한을 다 줄것입니다 


```

public UserBuilder roles(String... roles) {

	List<GrantedAuthority> authorities = new ArrayList<>(roles.length);
	for (String role : roles) {
		Assert.isTrue(!role.startsWith("ROLE_"),
				() -> role + " cannot start with ROLE_ (it is automatically added)");
		authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
	}
	return authorities(authorities);
}

```
안에서 우리가 준 role 은 앞에 prefix ROLE_ 가 붙은채로 생기게 됩니다 



그러면 우리는 Admin 권한과 일반 유저를 하나 만들었습니다 그러면 그들이 각각 호출할 수 있는 페이지와 , 메서드를 분리해서 사용을 하겠습니다 

## SecurityFilterChain
```

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

	httpSecurity.authorizeRequests().antMatchers("/admin/*").hasRole("ADMIN");
	httpSecurity.authorizeRequests().antMatchers("/normalRequest/*").authenticated();


	return httpSecurity.build();

}

```

아마 조금 이전 버전 시큐리티를 배우신 분이라면 이런식으로 시큐리티 설정파일을 설정했을것이다 

## WebSecurityConfigurerAdapter

```
public class SecurityConfig extends WebSecurityConfigurerAdapter  

```

이런식으로 WebSecurityConfigurerAdapter 를 상속받아서 사용했을것이다 물론 WebSecurityConfigurerAdapter 받아서 사용해도 되지만 2.7.1 버전 부터는 @Deprecated 
사용을 권장하고 있지 않는 모습이다 이런 이유는 아마 보안상 , 그리고 더 좋은 방식으로 httpSecurity 를 설정할 수 있기 때문일것입니다 물론 이 버전을 사용해서 구현을 해도 문제는 없지만 
결국 시큐리티 버전이 올라가면 갈 수록 사용을 권장하지는 않는것입니다 

## SecurityFilterChain

```
package com.cybb.main.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig{

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

        httpSecurity.authorizeRequests().antMatchers("/admin/*").hasRole("ADMIN");
        httpSecurity.authorizeRequests().antMatchers("/normalRequest/*").authenticated();


        return httpSecurity.build();

    }

    @Bean
    public UserDetailsService createUser(){
        UserDetails user = User.builder()
                                .username("user")
                                .password(passwordEncoder().encode("1234567890"))
                                .authorities("Read")
                                .roles("USER").build();

        UserDetails admin_user = User.builder()
                                .username("admin")
                                .password(passwordEncoder().encode("1234567890"))
                                .authorities("Read" , "Write" , "Update" , "Delete" )
                                .roles("ADMIN").build();


        return new InMemoryUserDetailsManager(user , admin_user);
    }
}

```
시큐리티는 SecurityFilterChain 을 사용하라고 권장을 하고 있습니다 그래서 저는 앞으로 SecurityFilterChain 으로 시큐리티 설정을 할것입니다 


## authorizeRequests , antMatchers , hasRole
```

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

	httpSecurity.authorizeRequests().antMatchers("/admin/*").hasRole("ADMIN");
	httpSecurity.authorizeRequests().antMatchers("/normalRequest/*").authenticated();

	return httpSecurity.build();
}

```
용어정리가 중요합니다 SecurityFilterChain 하나로 시큐리티 설정을 좌지우지 하는것이기 때문에 여기 설정에 있는 모든것들은 다 중요하고 앞으로 이에 대해서 설명을 이어 나갈것입니다
SecurityFilterChain 에 대해서는 다루지 않겠습니다 안의 내용은 인터페이스 이고 하단에 구현체들이 각각 구현을 하고 있어서 모든 내용을 다룰수 없습니다 앞부분에서 설명을 드렸다 싶히  
이 부분이 시큐리티의 핵심 엔진 부분입니다 


authorizeRequests -> 이 안에는 함축적인 단어인데 인증요청이라는 뜻입니다  
antMatchers -> ant 표현식으로 표현된 request 의 표현식이며 
hasRole -> 이러한 권한을 가지고 있어야 합니다 


authorizeRequests  + antMatchers + hasRole = 특정한 요청에는 반드시 특정한 role 를 가지고 있어야 합니다 


즉 지금 설정에서는 user 는 /admin 아래에 있는 모든 요청에 대해서는 거절당하게 됩니다 그럼 테스트를 해보시죠 
## admin 만 접근하는 핸들러 

```

package com.cybb.main.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("admin")
public class AdminController {
    
    @GetMapping("/")
    public String adminPage(){
        
        return "여기는 admin 페이지입니다";
    }
}


```

그럼 간단한 핸들러를 만들겠습니다 그리고 실행을 해서 user 와 admin 으로 각각 접근해보겠습니다 그리고 실행을 하면 문제가 있습니다 로그인페이지가 나오지 않게됩니다 

```
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception

```

우리가 이렇게 설정을 하는거는 시큐리티가 제공하는 기본적인 설정을 완전 뭉게겠다는 뜻입니다 그래서 로그인 페이지도 날라가서 나오지 않는데 이때는 하단에 

```
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

	httpSecurity.authorizeRequests().antMatchers("/admin/*").hasRole("ADMIN");
	httpSecurity.authorizeRequests().antMatchers("/normalRequest/*").authenticated();

	httpSecurity.formLogin();


	return httpSecurity.build();

}

```

이렇게 formLogin 을 사용하겠다고 적으면 기존처럼 로그인 화면이 나오게 됩니다 즉 깡그리 무시보다는 다시 호출하면 기본적인 설정은 할 수 있게 제공이 된다는 뜻입니다 

## admin user 으로  각각 로그인 

![admin 로그인](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/db644bcc-087f-4ac3-ac66-318c602cb251)

admin 유저로 로그인을 하게 되면 우리가 원하는데로 나오게 됩니다 다만 

![user 로그인](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/48c667ac-88a6-418e-9f7e-4dbded8db978)

다만 user 로 로그인을 하게 되면 이와 같이 403 에러가 발생하게 됩니다 이때 403 의 의미를 알아야 하는데 403은 Forbidden 금지됨 이라는 뜻으로 요청권한에 맞지 않은 사람이기 때문에 이에 대한 요청을 거절한다는 뜻입니다 즉 Role 이라는 권한으로 일반 사용자가 보는 페이지와 , 관리자가 보는 페이지를 분리할 수 있습니다 

## Authorities 

authorites 는 다음과 같은 경우에 쓸 수 있습니다 user 은 Read 만 가능하고 admin 은 Read , Write , Update , Delete 를 할 수 있습니다 이때 메서드에 다음과 같이 설정을 하게 되면 

```

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

	httpSecurity.authorizeRequests().antMatchers("/admin/*").hasRole("ADMIN")
                                    .antMatchers("/**/read").hasAuthority("Read")
                                    .antMatchers("/**/write").hasAuthority("Write");

	httpSecurity.formLogin();

	return httpSecurity.build();

}

```

새로운 두줄을 추가했습니다 예를 들어서 앞에서 어떤 요청이 오던간에 /read/** 가 중간에 들어가면 Read 권한만 접근이 가능하고 
마찬가지로 /write/** Write 권한만 접근이 가능하게끔 설정을 했습니다 이때는 user 권한으로 아래의 두개의 controller 을 호출하게 되면 

```

package com.cybb.main.controller;


import org.springframework.security.access.annotation.Secured;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("normalRequest")
public class NormalController {

    @GetMapping("/read")
    public String read(){

        return "read";
    }

    @GetMapping("/write")
    public String write(){

        return "write";
    }
}


```
이렇게 user 입장에서는 read 는 접근이 가능하지만 write 는 접근이 불가능합니다 

![read 접근](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0d738687-7e05-42b7-b0d5-0c35fba2a663)

read 로 호출해서 접근할때는 정상적으로 보이지만 

![write 접근](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/aedf3b8b-05b7-4fb4-b2e1-3a6ac93f6eba)

write 을 호출해서 접근할때는 보이지 않습니다 

오늘은 이렇게 Role 과 Authority 에 대해서 공부를 해보았습니다 





