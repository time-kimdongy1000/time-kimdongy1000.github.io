---
title: Spring MVC @SessionAttribute
author: kimdongy1000
date: 2023-04-14 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC , '@SessionAttribute' ]
math: true
mermaid: true
---

## @SessionAttribute 정의
@SessionAttribute 는  MVC 에서 세션에 데이터를 저장하고 컨트롤러에서 사용할 수 있도록 하는 애노테이션입니다 가장 흔한 사용방법은 
사용자가 웹어플리케이션에 로그인으 하면 그 로그인 정보를 세션에 저장하고 다른 페이지에서도 이 정보를 사용할 수 있도록 하는 애노테이션입니다 

이번시간에도 JPA 를 이용하여 간단한 회원가입과 , 로그인 을 활용해서 @SessionAttribute 활용하는 방법에 대해서 알아보겠습니다 

## 사전작업


maven
```

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-jpa -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-jdbc -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/com.h2database/h2 -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.1.214</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.24</version>
    <scope>provided</scope>
</dependency>

```
참고로 lombok 는 getter , setter 자동으로 만들어줘서 코드의 길이를 줄여주는것으로 앞으로 계속 채용하겠습니다
이번에도 클라이언트는 페이지는 따로 만들지 않으며 postman 으로 테스트를 진행하겠습니다 



UserController.java
```
package com.cybb.main.controller;

import com.cybb.main.dto.UserDto;
import com.cybb.main.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/createUser")
    public String createUser(@RequestBody UserDto userDto , RedirectAttributes redirectAttributes)
    {
        try{

            Long id = userService.createUser(userDto);

            redirectAttributes.addFlashAttribute("id" , id);

            return "redirect:/getUser";

        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }

    @GetMapping("/getUser")
    public ResponseEntity<UserDto> getUser(Model model)
    {

        try{
            Long id = (Long) model.asMap().get("id");

            UserDto userDto = userService.getUser(id);

            return new ResponseEntity<>(userDto , HttpStatus.OK);

        }catch(Exception e){
            throw new RuntimeException(e);
        }
    }

}


```
회원가입 controller 입니다 회원가입을 진행하면 redirect 로 getUser 를 호출해서 새로생긴 유저를 그대로 반환하는 로직입니다 


UserService.java
```

package com.cybb.main.service;

import com.cybb.main.dto.UserDto;
import com.cybb.main.entity.UserEntity;
import com.cybb.main.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Optional;


@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;


    public Long createUser(UserDto userDto) {

        UserEntity userEntity = UserEntity.builder()
                .username(userDto.getUsername())
                .password(userDto.getPassword())
                .age(userDto.getAge())
                .build();

        Long id =  userRepository.save(userEntity).getId();

        return id;
    }

    public UserDto getUser(Long id) {

        Optional<UserEntity> opUserEntity = userRepository.findById(id);

        if(opUserEntity.isPresent()){

            UserEntity userEntity = opUserEntity.get();

            return UserDto.builder()
                    .id(userEntity.getId())
                    .username(userEntity.getUsername())
                    .password(userEntity.getPassword())
                    .age(userEntity.getAge())
                    .build();

        }else{
            throw new RuntimeException("유저가 없습니다.");
        }
    }
}


```
UserService 입니다 회원가입시 사용되는 user 를 관리 (생성 , 조회 하는 서비스 로직입니다)

UserEntity.java
```

package com.cybb.main.entity;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String username;

    private String password;

    private String age;
}


```
UserEntity 는 이렇게 되어 있으며 UserDto 도 똑같은 필드로 작성되면됩니다 이때 롬북 라이브러리로 작성되어서 빌더패턴으로 작성이 되었습다 

![1](https://user-images.githubusercontent.com/58513678/231946089-98008cc4-f204-4be5-80a7-68f4937a1d6b.jpg)

이렇게 회원가입요청을 넣으면 그대로 회원정보를 만들어 냅니다 여기 까지 사전작업 

## 본코드 
@SessionAttribute 로 로그인시 로그인 정보를 세션단에서 저장을 하며 다른 controller 단에서 호출할때 유저정보를 같이 보내주는 형태가 될것입니다 

```

package com.cybb.main.controller;

import com.cybb.main.dto.UserDto;
import com.cybb.main.service.LoginService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.SessionAttributes;

import java.util.HashMap;
import java.util.Map;

@Controller
@SessionAttributes("loginUser")
public class LoginController {

    @Autowired
    private LoginService loginService;

    @PostMapping("/login")
    public String login(@RequestParam String username , @RequestParam String password , Model model)
    {
        try{

            Map<String , Object> param = new HashMap<>();
            param.put("username" , username);
            param.put("password" , password);

            UserDto userDto = loginService.login(param);
            model.addAttribute("loginUser" , userDto);

            return "redirect:/home";


        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}


```
로그인시에는 login 요청에 username , password 를 보내줍니다 그리고 클래스 상단에 @SessionAttributes("loginUser") 입력하면 이제 이제 이 클래스 안에서는 이 값을 공유하게 됩니다 
중요한것은 같은 클래스 안에서만 값을 공유하고 클래스 밖을 벗어난 곳에서는 값을 공유하지 않습니다 

```

@GetMapping("/home")
public ResponseEntity<UserDto> hello(@ModelAttribute("loginUser") UserDto userDto) {

    try {

        if (userDto == null) {
            throw new RuntimeException("로그인 유저가 없습니다");
        }

        return new ResponseEntity<>(userDto, HttpStatus.OK);


    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}

```
그래서 redirect 시켜주는 곳에서는 이와 같이 @ModelAttribute("loginUser") UserDto userDto 선언해서 바로 값을 불러와서 사용할 수 있습니다 
굳이 JPA 의 엔티티를 찾아서 값을 가져오지 않습니다.







