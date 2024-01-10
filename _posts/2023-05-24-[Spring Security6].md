---

title: Spring Secuirty 6 InMemoryUser
author: kimdongy1000
date: 2023-05-24 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true

---

지난시간에 로그인 유저를 가지고 올때 InMemory 데이터 베이스에 저장된것을 가지고 온것을 기억할것이다 가장 기본이 되는 저장소이지만 이는 휘발성이라 프로그램에 끊어지면 
없어지는 특징을 가집니다

## 새로운 유저 생성
```
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

SecurityConfig 이 클래스는 설정파일로서 시큐리와 관련한 설정을 이곳에서 할 예정입니다 
그리고 새로운 유저를 만들어볼텐데 우리는 username 은 time , 비밀번호는 1234567890 에 권한은 admin 을 가지는 user 를 생성해서 인메모리 유저에 넣어두도록 하겠습니다 
이제 우리가 새로운 User 를 만들게 되면 시큐리티가 제공하는 기본적인 User는 사용할 수 없게 됩니다 


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
이 부분이 시큐리티가 제공하는 가장 기본적인 User 관리입니다 