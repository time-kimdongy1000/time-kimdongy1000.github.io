---
title: Spring Secuirty 27 OAuth2 @AuthenticationPrincipal 
author: kimdongy1000
date: 2023-07-05 12:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true
---

우리는 지난시간 까지 정말로 길게 승인코드 발급부터 해서 access_token 을 발급받고 이 accesS_token 을 발급받아서 최종적으로 User 객체를 얻어와서 
시큐리티 컨텍스트에 심는거 까지 확인을 했다 그래서 이제 이걸 어떻게 쓸껀데로 돌아온것이다 이제 시큐리티 컨텍스트 안으로 들어온것이니 사실상 
사용자를 가져오는 방식은 1부 form 로그인하고 동일하다 소스를 보자


## @AuthenticationPrincipal
```

@RestController
public class DemoController {

    @GetMapping("")
    public String demoController(@AuthenticationPrincipal OAuth2User oAuth2User){


        return oAuth2User.toString();
    }
}


```

소스는 의외로 간단하다 결국 인증을 받게되면 시큐리티 컨텍스트에 유저 정보가 심어지게 되고 그것을 return 해서 웹화면으로 보게 되면 

```

Name: [30283228-fa36-4d88-82c3-c9494dc44bfd], Granted Authorities: [[ROLE_USER, SCOPE_email, SCOPE_profile]], User Attributes: [{sub=30283228-fa36-4d88-82c3-c9494dc44bfd, email_verified=false, name=time user, preferred_username=user1, given_name=time, family_name=user, email=user1@gmail.com}]

```

간단한 정보가 나오는것으로 끝이나게 된다 이제 우리는 로그인해서 유저정보를 가져와서 뿌리는거 까지 한바퀴를 돌아보았다 