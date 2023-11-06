---
title: Spring Secuirty 9 @AuthenticationPrincipal 
author: kimdongy1000
date: 2023-06-03 15:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

## 유저 설정과 필터설정

```
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder bcryptPasswordEncoder(){
        PasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder(10);
        return bCryptPasswordEncoder;
    }
    
    @Bean
    public SecurityFilterChain  securityFilterChain(HttpSecurity httpSecurity) throws Exception{
        
        httpSecurity.authorizeRequests().anyRequest().authenticated();

        httpSecurity.formLogin();
        
        
        return httpSecurity.build();
    }



    @Bean
    public UserDetailsService createUser(){

        UserDetails user = User.builder()
                                .username("user")
                                .password(bcryptPasswordEncoder().encode("1234567890"))
                                .authorities("Read")
                                .roles("USER").build();

        UserDetails admin_user = User.builder()
                                    .username("admin")
                                    .password(bcryptPasswordEncoder().encode("1234567890"))
                                    .authorities("Read" , "Write" , "Update" , "Delete" )
                                    .roles("ADMIN").build();

        return new InMemoryUserDetailsManager(user , admin_user);

    }
}
```

## @AuthenticationPrincipal
```
@RestController
public class DemoController {

    @GetMapping("/demo")
    public String demo (@AuthenticationPrincipal UserDetails userDetails){

        return userDetails.toString();
    }
}
```

간단한 DemoController 안에 핸들러를 작성을 해주는데 이때 @AuthenticationPrincipal 이다 이 애노테이션은 함축적인 애노테이션으로 인증된 유저의 시큐리티 Context 에 접근해서 UserDetails  에 접근해서 유저 정보를 가져오게 됩니다 그럼 웹에서는 이렇게 표현이 되는데 

`org.springframework.security.core.userdetails.User [Username=user, Password=[PROTECTED], Enabled=true, AccountNonExpired=true, credentialsNonExpired=true, AccountNonLocked=true, Granted Authorities=[ROLE_USER]]`

비밀번호는 민감정보로 보호가 되고 나머지는 현재 UserDetails 의 구현체에 맞게 정의가 되어 있는 모습입니다 또 다른 방식은 

```

@GetMapping("/demo2")
public String demo2 (Authentication authentication){

    UserDetails user = (UserDetails) authentication.getPrincipal();


    return user.toString();
}

```

이렇게도 가져올 수 있습니다 그럼 여기서 문제 과연 Authentication @AuthenticationPrincipal 는 중복이 되지 않을까요? 이게 이제 많은 사람들을 고민하게 되는중 하나인데 
결론부터 말하면 중복되지 않습니다 중복된다고 생각하는 사람은 다음과 같을 수 있습니다 핸들러를 여러명이 동시에 호출할때 라고 생각할 수 있지만 

애초에 AuthenticationPrincipal Authentication 은 자신만 접근할 수 있는 로컬쓰레드 중에서 보안컨텍스트에 저장이 됨으로 여러명이 동시에 같은 핸들러를 호출해도 
자신만 접근이 가능한 곳에서 저장된 데이터를 가져오기 때문에 다른 사람의 정보가 겹치거나 중복되지 않는 특징입니다 

이렇게 오늘은 인증된 유저를 호출하는 핸들러에서 가져오는 방법에 대해서 공부를 해보았습니다