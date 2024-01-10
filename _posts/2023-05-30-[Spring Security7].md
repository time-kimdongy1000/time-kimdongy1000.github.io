---
title: Spring Secuirty 7 PasswordEncoder
author: kimdongy1000
date: 2023-05-30 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---
우리는 지난시간에 InMermory 를 만들때 PasswordEncoder 를 사용했었는데 오늘은 이에 대해서 알아보도록 하겠습니다 


```
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Bean
    public UserDetailsService createUser(){
        UserDetails user = User.builder().username("time").password(passwordEncoder().encode("1234567890")).authorities("admin").build();
        return new InMemoryUserDetailsManager(user);
    }
}
```

## PasswordEncoder 정의
사용자의 비밀번호를 안전하게 저장하고 인증 시 비밀번호를 비교하기 위해 사용되는 Spring Security의 인터페이스입니다. 
비밀번호를 해싱(hashing)하고, 저장된 해시값과 사용자가 제공한 비밀번호를 비교하는 등의 작업을 수행합니다

이는 인터페이스로 이를 구현하는 구현체 몇가지를 소개하보자면 

1. BCryptPasswordEncoder
	강력한 해싱알고리즘을 사용하게 되는데 이때 salt 를 사용해서 좀더 복잡한 암호화를 진행할 수 있습니다 

2. NoOpPasswordEncoder
	실제로 비밀번호를 해싱하지 않고 있는 plan_text 그대로 저장하는 구현체입니다 보안측면에서는 위험수준이며 시큐리티는 이를 통해서 비밀번호를 저장하는것을 경고 처리하고 있습니다 

3. MessageDigestPasswordEncoder
	MessageDigest 우리가 잘 아는 SHA 방식으로 해싱을 하는 암호화 구현체입니다 

이 중에서 시큐리가 가장 추천하는 암호화 방식은 BCryptPasswordEncoder 를 채택하고 있습니다 

## 해싱 (Hash Algorithm)
위에서 계속 Hash 알고리즘으로 했는데 이에 대해서 간략하게 설명을 하고 가자면 이는 단방향 (일방향) 함수 단방향 함수는 역으로 진행을 할 수 없으며 특정 입력값을 해싱 하게 되면 
고정된 길이의 해싱 데이터를 return 하는 특징이 있습니다 이 밖에도 

1. 단방향 
	단방향은 역으로 불가능합니다 이 말은 한번 해싱된 값을 다시 역으로 돌려서 원래의 텍스트로 만들 수 없습니다 

2. 고정크기 
	해싱은 입력된 텍스트의 길이와 상관없이 항상 고정된 길이를 return 하게 됩니다 이때 SHA-256 같은 경우는 항상 256 자리 즉 32바이트 = 256비트 이는 영문자 숫자 (1비트로 계산)

3. 고유성
	같은 입력값에는 같은 해싱 결과값이 도출하지만 조금이라도 다른 입력값에 대해서는 전혀 다른 해싱값이 도출됩니다 

4. 솔팅 
	BCryptPasswordEncoder 같은 경우 salt 값을 추가할시 같은 입력에 대해서도 같은 출력을 보장하지 않습니다 이때 비밀번호는 해싱할때 salt 값을 해싱에 첨부해서 다른 raw_password 를 검증할떄 이 salt 값을 포함해서 검증을 진행하게 됩니다 


## BCryptPasswordEncoder
`public class BCryptPasswordEncoder implements PasswordEncoder`

그럼 우리가 유저를 생성할때 
`UserDetails user = User.builder().username("time").password(passwordEncoder().encode("1234567890")).authorities("admin").build();` 
이렇게 PasswordEncoder 를 호출해서 encode 를 하고 있음으로 디버깅을 잡아서 실행을 하게 되면 


```
public String encode(CharSequence rawPassword) {
    if (rawPassword == null) {
        throw new IllegalArgumentException("rawPassword cannot be null");
    }
    String salt = getSalt();
    return BCrypt.hashpw(rawPassword.toString(), salt);
}

```

이렇게 encode 함수를 호출하게 됩니다 이때 rawPassword 는 1234567890 들어오게 되고 만약 기초입력 비밀번호가 없으면 rawPassword cannot be null 을 던지고 끝이납니다 
그게 아니라면 이 salt 를 호출하게 되는데 salt 는 소금 여기서는 감미료 즉 단순히 사용자가 입력하는 비밀번호 말고 시큐리티는 복호화를 쉽게 못하게 감미료를 추가합니다 
이런 감미료를 추가하는 이유는 이미 BCryptPasswordEncoder 인코더의 알고리즘 자체는 노출이 되어 있기에 단순한 평문으로 인코딩을 하게 되면 레인보우 테이블 공격에 취약하게 됩니다 

## 레인보우 테이블 공격
이러한 공격방식은 해시함수를 사용해서 암호화 된 것들을 평문으로 복호화할려는 공격방식중에 하나로 이미 각 평문에 대한 암호문을 전부 테이블에 저장을 해두고 해싱문자열을 검색해서 해당 원문을 찾아내는 공격방식을 말합니다 이런것들이 가능한 이유는 거의 대부분의 암호화 알고리즘은 노출이 되고 어떻게 만들어지는지 다 알려졌습니다 그래서 단순 평문과 해싱을 이용한 사이트는 이런 공격으로 인해서 취약점을 발견하고 사용자의 계정을 탈취하는 것입니다 

## Salt 
그럼 시큐리티는 어떤 방어자세로 나가는가 하면 평문에 시큐리티가 랜덤으로 생성하는 약간의 감미료로 비밀번호를 첨가하게 됩니다 예를 들어서 1234567890 으로 초기 비밀번호를 만든다면시큐리티는 이곳에 ABCDEFG 를 추가해서 비밀번호는 1234567890ABCDFG 가 되는것이죠 이를 인코딩 하게 되면 해싱결과가 1234567890 을 해싱한것과 완전히 달라지게 됨으로 레인보우 테이블 공격은 막을 수 있습니다 그래서 시큐리티는 encode 를 할떄 Salt 를 추가해서 암호화 하게 됩니다 

```
@Override
public String encode(CharSequence rawPassword) {
    if (rawPassword == null) {
        throw new IllegalArgumentException("rawPassword cannot be null");
    }
    String salt = getSalt();
    return BCrypt.hashpw(rawPassword.toString(), salt);
}

private String getSalt() {
    if (this.random != null) {
        return BCrypt.gensalt(this.version.getVersion(), this.strength, this.random);
    }
    return BCrypt.gensalt(this.version.getVersion(), this.strength);
}
```

바로 하단에 salt 를 만드는 작업대가 있습니다 그럼 이런 salt 를 반환받으면 대략 $2a$10$wuT3P6I7eMQqTDvJR3exBe 이런 데이터가 반환이 들어오게 됩니다 
그럼 이제 return BCrypt.hashpw(rawPassword.toString(), salt); 평문과 salt 를 활용해서 완전 새로운 암호문을 만들게 됩니다 

## hashpw
```
private static String hashpw(byte passwordb[], String salt, boolean for_check) {
		BCrypt B;
		String real_salt;
		byte saltb[], hashed[];
		char minor = (char) 0;
		int rounds, off;
		StringBuilder rs = new StringBuilder();

		if (salt == null) {
			throw new IllegalArgumentException("salt cannot be null");
		}

		int saltLength = salt.length();

		if (saltLength < 28) {
			throw new IllegalArgumentException("Invalid salt");
		}

		if (salt.charAt(0) != '$' || salt.charAt(1) != '2') {
			throw new IllegalArgumentException("Invalid salt version");
		}
		if (salt.charAt(2) == '$') {
			off = 3;
		}
		else {
			minor = salt.charAt(2);
			if ((minor != 'a' && minor != 'x' && minor != 'y' && minor != 'b') || salt.charAt(3) != '$') {
				throw new IllegalArgumentException("Invalid salt revision");
			}
			off = 4;
		}

		
		if (salt.charAt(off + 2) > '$') {
			throw new IllegalArgumentException("Missing salt rounds");
		}

		if (off == 4 && saltLength < 29) {
			throw new IllegalArgumentException("Invalid salt");
		}
		rounds = Integer.parseInt(salt.substring(off, off + 2));

		real_salt = salt.substring(off + 3, off + 25);
		saltb = decode_base64(real_salt, BCRYPT_SALT_LEN);

		if (minor >= 'a') {
			passwordb = Arrays.copyOf(passwordb, passwordb.length + 1);
		}

		B = new BCrypt();
		hashed = B.crypt_raw(passwordb, saltb, rounds, minor == 'x', minor == 'a' ? 0x10000 : 0, for_check);

		rs.append("$2");
		if (minor >= 'a') {
			rs.append(minor);
		}
		rs.append("$");
		if (rounds < 10) {
			rs.append("0");
		}
		rs.append(rounds);
		rs.append("$");
		encode_base64(saltb, saltb.length, rs);
		encode_base64(hashed, bf_crypt_ciphertext.length * 4 - 1, rs);
		return rs.toString();
	}
```

이렇게 같은 페이지 hashpw 함수를 호출하면서 만드는데 이에 대한 분석은 하지 않겠습니다 그러면 사용자는 이제 암호화된 암호문을 받게 됩니다 그렇게 암호문을 저장받게 되고 이제 검증할때는 match 함수를 호출해서 비밀번호를 검증하게 됩니다 

## matches
```
public boolean matches(CharSequence rawPassword, String encodedPassword) {

if (rawPassword == null) {
    throw new IllegalArgumentException("rawPassword cannot be null");
}
if (encodedPassword == null || encodedPassword.length() == 0) {
    this.logger.warn("Empty encoded password");
    return false;
}
if (!this.BCRYPT_PATTERN.matcher(encodedPassword).matches()) {
    this.logger.warn("Encoded password does not look like BCrypt");
    return false;
}
return BCrypt.checkpw(rawPassword.toString(), encodedPassword);
}

```

비밀번호 검증은 이 matches 함수를 호출해서 원문하고 인코딩된 비밀번호를 비교하게 됩니다 그리고 안에서 salt 를 가져와서 비교하게 됩니다 그러면 여기서 질문하나가 생기게 되는데 salt 를 넣어준것이 아닌데 어떻게 salt 값을 알고 있냐? 사실 salt 는 반환되는것이 아니라 해싱된 데이터 안에 salt 가 포함이 되어 있음으로 이를 다시 salt 값과 인코딩된 값을 분리해서 비교를 하게됩니다 

이때 passwordEncoder 의 중요한 비교방식은 암호문을 디코딩해서 원문하고 비교하는것이 아닙니다 해싱은 단방향이기 때문에 역으로 가는것이 불가능 그렇기에 입력되는 비밀번호를 같은 해싱함수로 해싱을 해서 비교를 하는 방식으로 비교합니다 이때 일치하면 true , 다르면 false 를 반환하는 것입니다 