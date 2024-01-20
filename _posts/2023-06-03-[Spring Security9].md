---
title: Spring Secuirty 9 @AuthenticationPrincipal 
author: kimdongy1000
date: 2023-06-03 15:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

## 인증된 유저 사용
그럼 인증된 유저는 어떤 방식으로 참조를 걸 수 있을까 이떄 사용가능한게 바로 @AuthenticationPrincipal 입니다



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
@AuthenticationPrincipal 는 스프링 시큐리티에서 제공하는 애노테이션중 하나로 현재 사용자의 Principal 객체를 주입받을 때 사용할 수 있습니다 이를 지금처럼 Controller 메서드에서 참조해서 사용하면 인증된 객체를 컨트롤러 단위에서 사용할 수 있습니다 

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