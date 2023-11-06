---
title: Spring Secuirty 11 나만의 로그인 및 회원가입 만들기 2 
author: kimdongy1000
date: 2023-06-04 16:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security ]
math: true
mermaid: true
---

## csrf form 추가 
```
<h1 class="h3 mb-3 fw-normal"> 회원가입 </h1>

        <div class="form-floating">
            <input type="email" class="form-control" id="floatingInput" placeholder="name@example.com">
            <label for="floatingInput">Email address</label>
        </div>
        <div class="form-floating">
            <input type="password" class="form-control" id="floatingPassword" placeholder="Password">
            <label for="floatingPassword">Password</label>
        </div>
        <div class="form-floating">
            <input type="password" class="form-control" id="floatingPassword_confirm" placeholder="Password_confirm">
            <label for="floatingPassword_confirm">Password_confirm</label>
        </div>
        <input  th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
```

회원가입 제일 마지막 칸에 다음과 같이 csrfFilter 를 방지하는 태그를 하나 달아둘것이다 

```
<input  th:name="${_csrf.parameterName}" th:value="${_csrf.token}">

```

## CSRF 란 

CSRF(Cross-Site Request Forgery) 토큰은 웹 애플리케이션의 보안을 강화하는 데 사용되는 토큰종류이다 그럼 csrf 공격은 웹사이트 취약점을 이용한 공격의 한가지 방법입니다 

1) 선량한 사용자가 로그인을 하여 정당한 권한을 부여 받습니다 

2) 선량한 사용자는 악의적 사용자가 심어둔 공격페이지를 자신도 모르게 열게됩니다 (게시글 , 메일 등등 )

3) 공격자는 사용자가 공격페이지를 열람해서 얻은 쿠키 또는 세션을 얻게 됩니다 

4) 공격자는 이러한 세션 또는 쿠키를 활용해서 사이트를 공격합니다 이때 서버는 이러한 요청을 선량한 사용자가 요청을 얻은것이라고 판단해서 이를 실행하게 됩니다 

5) 선량한 사용자는 자신도 모르게 공격을 가담하게 된 가담자가 됩니다 

이런 공격을 막기 위한 방법중 하나가 csrf 토큰입니다 이 토큰이 없는 곳에서 요청을 보낼시 시큐리티는 악의적 공격이라고 판단한 뒤 이를 거부해버립니다 
이는 antMatchers , permit 의 여부와 상관 없이 서버의 상태를 변경하는 (post , put , delete , patch ) 메서드를 사용하는 곳에서 이 csrf 가 발급한 토큰을 넣지 않고 요청을 보낼때 거절당하게 됩니다 그럼 이 토큰 발급과 어떻게 인증이 되는지는 끝에서 알아보도록 하겠습니다 


## 서버 - 클라이언트 통신준비 
```
<script src="/resources/js/register.js"></script>
```

## register.js 작성 
```

const floatingInput = document.querySelector("#floatingInput");                 //아이디
const floatingPassword = document.querySelector("#floatingPassword");           // 비밀번호
const floatingPassword_confirm = document.querySelector("#floatingPassword_confirm");   //비밀번호 확인
const btn_register = document.querySelector("#btn_register");
const csrf_input = document.querySelector('input[name="_csrf"]');           //csrf  토큰

btn_register.addEventListener('click' , (e) => {
    
    const username = floatingInput.value;
    const password = floatingPassword.value;
    const confirm_password = floatingPassword_confirm.value;
    const csrf_token = csrf_input.value;


    if(!username){
        alert('아이디를 입력해주세요.')
        return;
    }


    if(!password){
        alert('비밀번호를 입력해주세요.')
        return;
    }

    if(!confirm_password){
        alert('비밀번호 확인을 입력해주세요.')
        return;
    }

    if(password !== confirm_password){
        alert('비밀번호가 같지 않습니다.')
        return;
    }

    const domain = "http://localhost:8080"
    const url = "/signUp/register"
    const method = "post";
    const data = {
        "username" : username ,
        "password" : password ,
        "confirm_password" : confirm_password
    }

    try{

        fetch(domain + url , {
            method : method ,
            headers: {
                "Content-Type": "application/json" ,
                "X-CSRF-Token" : csrf_token
            },

            body : JSON.stringify(data)
        })
        .then( (response) => response.json())
        .then ( (data) => {
            console.log(data);
        })

    }catch(error){
        console.error("실패 : " , error)
    }
})
```

## SignUpController 작성 
```
@Controller
@RequestMapping("signUp")
public class SignUpController {

    @Autowired
    private SignUpService signUpService;

    @GetMapping("/")
    public String signUpPage(){

        return "signUp";
    }

    //회원가입 하는 곳
    @PostMapping("/register")
    public ResponseEntity<?> registerUser(@RequestBody UserDto userDto)
    {

        try{

            Long id = signUpService.registerUser(userDto);
            UserDto save_user = signUpService.findByUserId(id);


            return new ResponseEntity<>(save_user , HttpStatus.OK);
        }catch (Exception e){
            throw new RuntimeException(e);
        }


    }
}
```

## SignUpService 작성
```
@Service
public class SignUpService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private UserRepository userRepository;

    public Long registerUser(UserDto userDto) {

        if(!StringUtils.hasText(userDto.getUsername())){
            throw new RuntimeException("username 은 null 이 될 수 없습니다.");
        }

        if(!StringUtils.hasText(userDto.getPassword())){
            throw new RuntimeException("password 은 null 이 될 수 없습니다.");
        }

        if(!StringUtils.hasText(userDto.getConfirm_password())){
            throw new RuntimeException("confirm_password 은 null 이 될 수 없습니다.");
        }

        if(!userDto.getConfirm_password().equals(userDto.getPassword())){
            throw new RuntimeException("비밀번호는 서로 다를 수 없습니다.");
        }

        UserEntity userEntity = new UserEntity();
        userEntity.setUsername(userDto.getUsername());
        userEntity.setPassword(passwordEncoder.encode(userDto.getPassword()));

        return userRepository.save(userEntity).getId();
    }

    public UserDto findByUserId(Long id) {

        Optional<UserEntity> userEntity = userRepository.findById(id);

        if(userEntity.isPresent()){
            UserDto userDto = new UserDto();
            userDto.setUsername(userEntity.get().getUsername());
            userDto.setPassword(userEntity.get().getPassword());

            return userDto;

        }else{
            throw new RuntimeException("username 이 존재하지 않습니다.");
        }
    }
}
```

## UserDto
```

package com.cybb.main.dto;

public class UserDto {

    private String username;
    private String password;

    private String confirm_password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getConfirm_password() {
        return confirm_password;
    }

    public void setConfirm_password(String confirm_password) {
        this.confirm_password = confirm_password;
    }
}
```

## UserEntity
```
@Entity
@Table(uniqueConstraints =  {@UniqueConstraint(columnNames = "username" )})
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;

    private String password;

    public UserEntity(Long id, String username, String password) {
        this.id = id;
        this.username = username;
        this.password = password;
    }

    public UserEntity() {

    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```


## UserRepository
```
@Repository
public interface UserRepository extends JpaRepository<UserEntity , Long> {
}
```
## 설명 

그냥 평범한 클라이언트 서버 통신 코드입니다 다른점은 csrf 로직이 심겨졌다는 것고 물론 이는 Securrity 의 Filter 요청이기에 다음장에서 다루는것으로 하고 
평함한 mvc 패턴으로 db 에 저장을 하는 로직입니다 저장할떄 Entity 라는 것을 이용해서 인메모리 DB 에 저장을 한 상태이고 

DB 에 저장을 할떄 password 는 encode 해서 저장하고 있습니다 

특이한점은 잠시 UserEntity 보게 되면 JAP 를 다루진 않지만 @Entity 애노테이션이 붙게 되면 이 테이블이 생기게 되고 JPA 객체가 생겨나게 됩니다 
```
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Table(uniqueConstraints =  {@UniqueConstraint(columnNames = "username" )})
```

이때 자동생성 Key는 Long 타입으로 잡고 이떄 `@GeneratedValue(strategy = GenerationType.IDENTITY)` 자동으로 생성하는 value 인데 이때 생성방식 타입을 설정할 수 있습니다 그리고 `@Table(uniqueConstraints =  {@UniqueConstraint(columnNames = "username" )})` 는 컬럼중에 username 이라는 컬럼이 있는데 이 username 을 유니크 key 로 설정해서 중복 username 이 엔티티에 insert 되지 않도록 하는것입니다 이를 RDBMS 로 대입 해보면

mysql 기준으로 @GeneratedValue == auto increment 가 되는것이고 @UniqueConstraint 는 테이블을 만들때 제약조건 중에 Unique 를 설정하는 부분입니다 


UserRepository 는 좀 특이하지만 JpaRepository 상속받음으로써 굳이 적지 않아도 부모가 가지고 있는 일정한 메서드를 바로 사용할 수있습니다 
대표적으로 save (insert) , findbyId (select) 이렇게 구성이 되어 있습니다 그러면 우리는 이제 웹화면으로 가서 가입을 해보면

![csrf_token](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/37d4e9f4-9a9f-4306-bfab-e75a1059fb0c)

이제 하단 csrf_input 이 생겼습니다 실제라이브 시스템에는 이를 당연히 hidden 처리 해서 사용자 눈에는 보이지 않게끔 처리해야 합니다만 우리는 눈에 보이고 설명을 하는것이 좋으습니다만 지금은 그냥 보이는 채로 진행을 하겠습니다 


![회원가입](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/16662694-6ab1-4d26-9823-522750aeb637)

회원가입을 하게 되면 이렇게 옆에 console 에 뜨는데 저는 같은 username 을 두번 보내서 한번은 성공해서 저장하고 다른 한번은 제약조건이 발생되어서 
fail 하는 모습을 보여드린것입니다 우리는 이제 DB 에 저장을 한 거 까지는 완료 되었습니다 그리고 return 되는 비밀번호는 원래 보여서는 안되지만 현재 인코딩 되어서 잘 저장된 모습을 볼 수 있습니다