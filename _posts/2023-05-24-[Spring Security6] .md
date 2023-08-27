---

title: Spring Secuirty 6 InMemoryUSer
author: kimdongy1000
date: 2023-05-24 10:00
categories: [Spring, Security]
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

그 이유가 뭐냐 우리 앞에서 곰곰히 생각을 해보면 유저를 생성할때 비밀번호 인코딩 여부를 판별하는 로직이 있었습니다 비밀번호 인코딩 흔적이 없으면 바로 에러를 던지는 로직을 호출하게 되어서 콘솔은 이런 에러가 생기게 됩니다  There is no PasswordEncoder mapped for the id "null"

passwrodEncoder 이 null 인거 같은데 하면서 에러를 던지고 끝이나게 됩니다 즉 시큐리티 보안에서는 비밀번호는 무조건 passwordEncoder 를 거쳐야합니다 
그래서 우리는 passwrodEncoder 에 대해서 알아볼것인데 대칭키 하나 비대칭키 하나에 대해서 알아보도록 하겠습니다 

## PasswordEncoderFactories

```

@SuppressWarnings("deprecation")
public static PasswordEncoder createDelegatingPasswordEncoder() {
	String encodingId = "bcrypt";
	Map<String, PasswordEncoder> encoders = new HashMap<>();
	encoders.put(encodingId, new BCryptPasswordEncoder());
	encoders.put("ldap", new org.springframework.security.crypto.password.LdapShaPasswordEncoder());
	encoders.put("MD4", new org.springframework.security.crypto.password.Md4PasswordEncoder());
	encoders.put("MD5", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("MD5"));
	encoders.put("noop", org.springframework.security.crypto.password.NoOpPasswordEncoder.getInstance());
	encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
	encoders.put("scrypt", new SCryptPasswordEncoder());
	encoders.put("SHA-1", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-1"));
	encoders.put("SHA-256",
			new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-256"));
	encoders.put("sha256", new org.springframework.security.crypto.password.StandardPasswordEncoder());
	encoders.put("argon2", new Argon2PasswordEncoder());
	return new DelegatingPasswordEncoder(encodingId, encoders);
}

```

## 비대칭키
스프링 시큐리티가 제공하는 암호화 모듈종류이다 현재 시큐리가 권장하고 있는 암호화 방식은 bcrypt 방식입니다 이는 비대칭키로 개인키 , 공개키 를 가지고 암호화 및 복호화를 합니다 원리는 이렇게 됩니다 

공개키는 말그래도 누구에게나 공개가 되어 있는 key 입니다 그리고 이런 공개키로 인코딩을 하면 서버는 개인키를 가지고 이를 복호화 하게 됩니다 그렇기에 
공개키는 노출되어도 문제가 없지만 개인키는 노출이 되면 보안상 문제가 발생하게 됩니다 

1. 키생성 비대칭키를 생성하면 공개키와 비공개키가 생성이 되고 공개키는 누군지 사용할 수 있게 보여지게 되고 비공개키(개인키) 서버에 보관이 됩니다 

2. 송신자는 공개키를 이용해서 인코딩 후 데이터를 서버로 전송 

3. 서버는 개인키를 가지고 인코딩 데이터를 복호화 합니다 이 과정에서 원래 송신자가 보냈던 데이터 원문이 드러나게 됩니다 

4. 서버는 이 원문을 가지고 비교를 해서 올바른지 아닌지 판별하게 됩니다 

## 대칭키 
대칭키는 데이터를 암호화 하고 복호화 하는데 하나의 key 만을 가지는것을 말합니다 그렇게 되면 이 key 는 비공개key 로 노출이 되면 수신자 , 송신자 사이에 전송되는 데이터를 탈취해서 원문을 분석 할 수 있게 됩니다 

1. 키생성 이때 key 는 개인키만 생성되므로 이때 key 는 수신자 , 송신자만 가지게 됩니다 

2. 송신자는 key 를 가지고 원문을 암호화 합니다 

3. 수신자는 key 를 가지고 송신자가 보낸 key 를 복호화 합니다 

언뜻 보면 대칭키 비대칭키 안전하게 보이지만 대칭키는 수신자 , 송신자 모두 암호화 , 복호화 할때 동일한 key 를 가지기에 둘중 한명이 key 를 잃어버리거나 탈취당하면 
보낸 원문을 읽지 못하거나 , 중간에 내용이 탈취당할 수 있습니다 

그에 반해 비대칭키는 송신자는 공개키로 데이터를 전송하기에 복호화 할때 사용하는 개인키는 수신자만 가지고 있어 수신자만 보관을 잘 하고 있으면 보안에 문제가 없습니다 


그래서 시큐리티는 비대칭키를 좀더 선호하는 방식으로 개발이 되었습니다 (그렇다고 대칭키가 안쓰이는것이 아닙니다) 한국 인터넷 진흥원에 따르면 비밀번호는 복호화가 되지 않는 단문으로 해싱이 되어야 한다고 적혀 있습니다 이는 대칭키 SHA 해싱알고리즘만 가능한 암호화 방식입니다 

그럼 각각 하나씩 암호화를 진행을 해보겠습니다 

## SHA-256 

```
package com.cybb.main.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.DelegatingPasswordEncoder;
import org.springframework.security.crypto.password.MessageDigestPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

import java.util.Map;

@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder sha_256PasswordEncoder(){
        PasswordEncoder sha_password_256 = new MessageDigestPasswordEncoder("SHA-256");
        return sha_password_256;
    }

    @Bean
    public UserDetailsService createUser(){

        UserDetails user = User.builder().username("time").password(sha_256PasswordEncoder().encode("12345678900")).authorities("admin").build();
        System.out.println(sha_256PasswordEncoder().encode("1234567890"));
        return new InMemoryUserDetailsManager(user);

    }


}


```

이렇게 SHA-256 알고리즘을 활용해서 암호화 하게 됩니다 그럼 잠시 암호화 과정을 살펴보면


## MessageDigestPasswordEncoder
```

MessageDigestPasswordEncoder


@Override
public String encode(CharSequence rawPassword) {
	String salt = PREFIX + this.saltGenerator.generateKey() + SUFFIX;
	return digest(salt, rawPassword);
}

private String digest(String salt, CharSequence rawPassword) {
	String saltedPassword = rawPassword + salt;
	byte[] digest = this.digester.digest(Utf8.encode(saltedPassword));
	String encoded = encode(digest);
	return salt + encoded;
}

private String encode(byte[] digest) {
	if (this.encodeHashAsBase64) {
		return Utf8.decode(Base64.getEncoder().encode(digest));
	}
	return new String(Hex.encode(digest));
}


```

이때 salt 는 원문을 빼고 교란(?) 원문을 쉽게 추측할 수 없게 추가하는 약간의 감미료같은 것입니다 매번 실행할떄마다 특정해주지 않으면 스스로 값을 만들어 내고 있으며 
`String saltedPassword = rawPassword + salt; byte[] digest = this.digester.digest(Utf8.encode(saltedPassword));`
이 두 메서드 때문에 감미료가 첨가된 1234567890 는 복호화 되어서 쉽게 추측 할 수 없게 됩니다 그렇게 우리는 대칭키 하나가 만들어졌고 

그렇기에 여기서는 추가적으로 설명을 하자면 `password(sha_256PasswordEncoder().encode("12345678900")` 값하고 
` System.out.println(sha_256PasswordEncoder().encode("1234567890"));` 서로 다른 값입니다 그 이유는 encode 를 호출할때마다 salt 값이 달라지기 때문입니다

비대칭키도 바로 알아보겠습니다 

## 비대칭키 

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
        System.out.println(bcryptPasswordEncoder().encode("1234567890"));
        return new InMemoryUserDetailsManager(user);

    }
}


```

마찬가지로 여기도 encode 를 살펴보면 

```

public String encode(CharSequence rawPassword) {
	if (rawPassword == null) {
		throw new IllegalArgumentException("rawPassword cannot be null");
	}
	String salt = getSalt();
	return BCrypt.hashpw(rawPassword.toString(), salt);
}

```

encode 를 호출할때마다 salt 감미료를 비밀번호에 첨가하게 되고 그것과 더불어서 BCrypt.hashpw(rawPassword.toString(), salt); 비밀번호를 해싱하게 됩니다 
그렇기에 역시나 마찬가지로 비밀번호를 설정할때 `User.builder().username("time").password(bcryptPasswordEncoder().encode("12345678900"))`
비밀번호와  System.out.println(bcryptPasswordEncoder().encode("1234567890")); 값은 서로 다르게 나오게 됩니다 

오늘은 우리가 특별하게 user 를 커스텀하게 만들어보았습니다 그리고 유저를 생성할때 passwordEncoder 와 현재 시큐리티가권장하는 passwordEncoder 을 살펴보았습니다 




