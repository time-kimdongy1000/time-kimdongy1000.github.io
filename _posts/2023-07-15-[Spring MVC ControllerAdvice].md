---
title: Spring ControllerAdvice
author: kimdongy1000
date: 2023-07-15 10:00
categories: [Back-end, Spring - MVC]
tags: [ MVC , ControllerAdvice ]
math: true
mermaid: true
---

웹을 개발하다보면 클라이언트에서 에러메세지를 작성해서 보여줄 수 있지만  서버에서 작성한 에러메세지를 클라이언트로 보내야 하는 경우가 있다 spring mvc 에서는 예외처리를
이렇게 하게 된다 

## 클라이언트
```

/*회원가입 로직*/

const url = "http://localhost:8080/signUp/"
const method = "post"
const data = {
    "email" : email ,
    "password" : password ,
    "confirm_password" : confirm_password
}

try{

    fetch(url , {
        "method" : method ,
        "headers" : {
            "Content-Type" : "application/json"
        } ,
        "body" : JSON.stringify(data)

    }).then( res => {

        console.log(res);

        return res.json();

    }).then( result => {

        console.log(result);
    })

}catch(error){


}


```

이 코드는 서버와 통신을 하기 위한 자바 스크립트 promise 객체로 signUp 곳으로 요청을 주게 되면 

## 백엔드 
```

@PostMapping(value =  "/" )
public ResponseEntity<?> signUp (@RequestBody SignUpDto signUpDto)
{

    try{

        signUpService.signUp(signUpDto);

        return null;
    }catch(Exception e){
        throw new RuntimeException(e.getMessage());
    }


}

```

이런식으로 작성이 되어 있고 signUp 코드가 진행함에 따라서 에러가 발생을 하면 throw new RuntimeException(e.getMessage()); 으로 전달이 되고 끝이 난다 
그럼 signUp 소스를 코드를 한번보게 되면 

```

public void signUp(SignUpDto signUpDto) {

    String email = signUpDto.getEmail();
    String password = signUpDto.getPassword();
    String confirm_password = signUpDto.getConfirmPassword();

    Matcher matcher = pattern.matcher(email);

    if(!StringUtils.hasText(email)) throw new RuntimeException("이메일이 존재하지 않습니다");
    if(!StringUtils.hasText(password) || !StringUtils.hasText(confirm_password)) throw new RuntimeException("비밀번호가 존재하지 않습니다");
    if(!password.equals(confirm_password)) throw new RuntimeException("비밀번호가 일치하지 않습니다");
    if(!matcher.matches()) throw new RuntimeException("이메일 형식이 아닙니다");
}

```

이렇게 각 상황에 맞추어서 throw new RuntimeException 에러를 발생시키는데 이때 구체적으로 어떤 에러가 발생되는지 적고 있다 만약 이렇게 해서 내가 에러를 한번 발생을 시키게 되면 

## 서버 에러 
```
java.lang.RuntimeException: 이메일이 존재하지 않습니다

```
우리가 보는 서버 에러에서는 이렇게 나오고 

## 클라이언트 에러 
```
error: "Internal Server Error"
path: "/signUp/"
status: 500
timestamp : "2023-11-28T12:45:53.256+00:00"

```
이렇게 에러가 발생한것은 알겠는데 구체적으로 어떤 에러가 발생했는지 알 수가 없다 이때 사용할 수 있는게 바로 ControllerAdvice 가 존재하는데 

## ControllerAdvice 정의
@ControllerAdvice 어노테이션은 전역 예외 처리를 담당하는 클래스로서 어떤 핸들러에서 에러가 발생시 최종적으로는 @ControllerAdvice 로 전달되어서 
커스텀하기 나름에 따라서 클라이언트에 적절한 메세지를 전달해줄 수 있습니다 

## ControllerAdvice 작성 
```
@ControllerAdvice
public class ExceptionControllerAdvice {

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity<?> handleException(Exception e) {

        Map<String , Object> resultMsg = new HashMap<>();
        resultMsg.put("errorMsg" , e.getMessage());

        return new ResponseEntity<>(resultMsg ,  HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

```
@ControllerAdvice 에노테이션은 그래서 @ExceptionHandler(Exception.class) 발생한 핸들러는 최종적으로 이 핸들러로 들어오게 되고 return 으로 우리가 만든 특정 에러 메세지를 전달할 수 있습니다 그러면 이제 이를 기동해서 에러를 일으키게 되면 back-end 로그는 아까와 마찬가지로 같은 모양의 에러가 발생하게 되는데 

```
{errorMsg: '이메일이 존재하지 않습니다'}
errorMsg: "이메일이 존재하지 않습니다"
```
이렇게 서버에서 작성한 에러메세지가 전달되게 됩니다 이