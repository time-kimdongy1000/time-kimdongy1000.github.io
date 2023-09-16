---

title: Spring Secuirty 6 InMemoryUser
author: kimdongy1000
date: 2023-05-24 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

우리는 지난시간에 인증과정 전체적인 부분을 한번 쭈욱 살펴보았다 그중에서 특히 user를 저장하고 있는 공간인 InMemoryUserDetailsManager 에 대해서 알게되었는데 오늘부터는 직업 우리가 커스텀하는 시간을 가지면서 필요한 시큐리티 공부를 해보자 

그럼 코드를 먼저 작성을 하겠습니다 

## 완전히 다른 유저 생성 
우리는 새로운 유저를 한명 만들어보겠습니다 

```

package com.cybb.main.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService createUser(){

        UserDetails user = User.builder().username("time").password("1234567890").authorities("admin").build();

        return new InMemoryUserDetailsManager(user);

    }


}


```
새로운 config 파일을 하나 만들어줄텐데 이름은 SecurityConfig 하고 여기에 시큐리티 설정에 관한 것들을 정리해서 나열하도록 하겠습니다 
그리고 새로운 유저를 만들어볼텐데 우리는 아이디는 time , 비밀번호는 1234567890 에 권한은 admin 을 가지는 user 를 생성해서 인메모리 유저에 넣어두도록 하겠습니다 
그럼 실행을 하게 되면 우리는 이제 기본적으로 시큐티에서 제공하는 유저는 사용할 수 없습니다 그럼 이 계정으로 이제 로그인을 하면 로그인이 안될것입니다

## PasswordEncoder 
```

package com.cybb.main.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.DelegatingPasswordEncoder;
import org.springframework.security.crypto.password.MessageDigestPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

import java.util.Map;

@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder bcryptPasswordEncoder(){
        PasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder(10);
        return bCryptPasswordEncoder;
    }

    @Bean
    public UserDetailsService createUser(){

        UserDetails user = User.builder().username("time").password(bcryptPasswordEncoder().encode("12345678900")).authorities("admin").build();
        return new InMemoryUserDetailsManager(user);

    }


}



```

PasswordEncoder 을 Bean 으로 삽입을 해주고 비밀번호를 인코딩을 해주겠습니다 PasswordEncoder 에 대해선 다음시간에 다뤄보는것으로 하고 이번시간에는 InMemoryUserDetailsManager 알아보도록 하겠습니다 

## User 
```
public class User implements UserDetails, CredentialsContainer 

private String password;  			// 비밀번호

private final String username;		// 아이디

private final Set<GrantedAuthority> authorities;	//권한

private final boolean accountNonExpired;		//계정 만료여부

private final boolean accountNonLocked;			// 계정 잠김여부

private final boolean credentialsNonExpired;	//비밀번호 만료여부

private final boolean enabled;					// 사용여부

```

로써 Builder 패턴으로 객체의 값을 집어넣을 수 있습니다 다형성의 원리에 의해서 UserDetails 를 구현하고 있기 때문에 
`UserDetails user = User.builder().username("time").password(bcryptPasswordEncoder().encode("12345678900")).authorities("admin").build();` 형식으로 코드를 적을 수 있습니다 그리고 안에 있는 필드들은 위와 같은데 저런 필드들이 있습니다 디폴트값이 아닌 username , password , authorities 를 제외하면 
기본적으로 제공되는 값이 존재합니다 boolean 타입들은 전부 false 값을 가집니다 

그리고 실행을 하게 되면 바로 아래 클래스가 호출이 되는데 이는 

## InMemoryUserDetailsManager
```
public class InMemoryUserDetailsManager implements UserDetailsManager, UserDetailsPasswordService 


public InMemoryUserDetailsManager(UserDetails... users) {
		for (UserDetails user : users) {
			createUser(user);
		}
	}


public void createUser(UserDetails user) {
		Assert.isTrue(!userExists(user.getUsername()), "user should not exist");
		this.users.put(user.getUsername().toLowerCase(), new MutableUser(user));
	}


```


메모리상에 User 를 생성하는것이기 때문에 기동과 동시에 유저를 생성하게 됩니다 이때 createUser 를 호출하게 되는데 이때 createUser 는 동일한 user 가 존재하지 않는지 확인을 하고 없으면 `private final Map<String, MutableUserDetails> users = new HashMap<>();` 이런 Map 형식의 타입인데 여기에 생성을 하게 됩니다 

그럼 생성된 user 를 반환하고 이를 사용할 수 있게 메모리게 user 를 올려고 놓고 로그인할때마다 메모리상에 접근해서 loadByUserName 으로 호출을 해왔습니다 
이 과정까지는 지난시간까지 우리가 쭈욱 해왔던 내용입니다 

그럼 오늘은 간단하게 새로운 유저를 만들고 로그인하고 이때 생성되는 유저가 어떤 형태인지 알아보았습니다 
