---
title: Spring Secuirty 8 Authorities , Role 
author: kimdongy1000
date: 2023-06-01 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

## Authorities , Role 의 각각 정의 

1. Role 정의 
	이는 주로 이 User 가 가지는 그룹을 나타냅니다 일반 유저를 나타내는 유저 그리고 관리자를 나타내는 Admin 등등 지정할 수 있으며 이는 유저의 컨셉과 사용자의 사용법에 따라서 다르게 정의 됩니다 


2. Authorities 정의 
	이는 주로 권한을 나타냅니다 읽기 권한 쓰기 권한 수정권한 삭제권한 Role 처럼 권한을 나누고 메서드 호출할때 해당 Authorities 를 지정하게 되면 해당 권한을 가지고 있지 않은 유저는 해당 메서드를 호출 할 수 없습니다 



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
안에서 우리가 준 role 은 앞에 prefix "ROLE_" 가 붙은채로 생기게 됩니다 



그러면 우리는 Admin 권한과 일반 유저를 하나 만들었습니다 그러면 그들이 각각 호출할 수 있는 페이지와 , 메서드를 분리해서 사용을 하겠습니다 

## SecurityFilterChain

```
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

        httpSecurity.formLogin();


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
용어정리가 중요합니다 SecurityFilterChain 하나로 시큐리티 설정을 좌지우지 하는것이기 때문에 여기 설정에 있는 모든것들은 다 중요하고 앞으로 이에 대해서 설명을 이어 나갈것입니다 SecurityFilterChain 에 대해서는 다루지 않겠습니다 안의 내용은 인터페이스 이고 하단에 구현체들이 각각 구현을 하고 있어서 모든 내용을 다룰수 없습니다 

authorizeRequests -> 이 안에는 함축적인 단어인데 인증요청이라는 뜻입니다  
antMatchers -> ant 표현식으로 표현된 request 의 표현식이며 
hasRole -> 이러한 권한을 가지고 있어야 합니다 


authorizeRequests  + antMatchers + hasRole = 특정한 요청에는 반드시 특정한 role 를 가지고 있어야 합니다 
즉 지금 설정에서는 user 는 /admin 아래에 있는 모든 요청에 대해서는 거절당하게 됩니다 그럼 테스트를 해보시죠 


## admin 만 접근하는 핸들러 

```
@RestController
@RequestMapping("admin")
public class AdminController {
    
    @GetMapping("/")
    public String adminPage(){
        
        return "여기는 admin 페이지입니다";
    }
}

```

## admin user 으로  각각 로그인 

![admin 로그인](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/db644bcc-087f-4ac3-ac66-318c602cb251)

admin 유저로 로그인을 하게 되면 우리가 원하는데로 나오게 됩니다 다만 

![user 로그인](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/48c667ac-88a6-418e-9f7e-4dbded8db978)

다만 user 로 로그인을 하게 되면 이와 같이 403 에러가 발생하게 됩니다 이때 403 의 의미를 알아야 하는데 403은 Forbidden 금지됨 이라는 뜻으로 요청권한에 맞지 않은 권한이기 때문에 이에 대한 요청을 거절한다는 뜻입니다 즉 Role 이라는 권한으로 일반 사용자가 보는 페이지와 , 관리자가 보는 페이지를 분리할 수 있습니다 

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

write 을 호출해서 접근할때는 보이지 않습니다 이렇게 User 의 적절한 권한 또는 Role 을 포함시키면 각 유저가 들어갈 수 있는 페이지와 , 들어갈 수 없는 페이지를 구성할 수 있게 됩니다 

