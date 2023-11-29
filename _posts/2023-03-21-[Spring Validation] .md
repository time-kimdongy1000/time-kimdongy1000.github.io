---
title: Spring Ioc Validation
author: kimdongy1000
date: 2023-03-21 14:20
categories: [Back-end, Spring - Core]
tags: [ Validation ]
math: true
mermaid: true
---

## 유효성 검사
Spring 이 추구하는 유효성 검사는 단순 웹계층에 국한되지 않고 어플리케이션 안에서 모든 영역에 대한 유효성을 처리할 수 있게끔 개발이 되었다 
그래도 보여주기엔 web 이 훨씬 나아서 web 프로젝트로 개발해서 한번 진행을 해볼려고 합니다 




## 유효성 검사 인터페이스
예를 들어서 다음과 같은 도메인 클래스가 있다고 하자

```
@Controller
public class CustomController {
    @Autowired
    private Users users;

    @PostMapping("/")
    @ResponseBody
    public CustomMessage insertUser(@RequestParam("name") String name , @RequestParam("age") int age)
    {

        CustomMessage customMessage = new CustomMessage();

        try{

            User user = new User();

            user.setUserId(users.getUsersList().size());
            user.setName(name);
            user.setAge(age);

            users.addUser(user);

            customMessage.setMessage("새로운 유저가 생성되었습니다.");
            customMessage.setDomain(user);

            return customMessage;

        }catch(Exception e){

            customMessage.setMessage("유저 생성이 실패했습니다");
            customMessage.setDomain(null);

            return customMessage;
            
        }
    }
}


```

```
public class User {

    private int userId;
    private String name;
    private int age;

    public int getUserId() {
        return userId;
    }

    public void setUserId(int userId) {
        this.userId = userId;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}


```

```
@Component
public class Users {

    private List<User> usersList;

    public Users(){
        this.usersList = new ArrayList<>();
    }

    public List<User> getUsersList() {
        return usersList;
    }

    public void addUser(User user){
        usersList.add(user);
    }
}

```

```
public class CustomMessage<T> {

    private String message;

    private T domain;

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public T getDomain() {
        return domain;
    }

    public void setDomain(T domain) {
        this.domain = domain;
    }
}

```

기초도메인은 이렇게 된다 그리고 예를 들어서 이곳에서 제약조건을 줘보자 가입조건은 name 은 빈값이 아닌 길이가 8 이상이어야 하고 
age 는 10 이상으로 세팅을 해야 한다 그럼 Controller 을 수정을 해보자

```

if(!StringUtils.hasText(name) && name.length() < 8){
    throw new RuntimeException("name 이 비어있거나 이름의 길이가 8보다 작습니다");
}

...

catch(Exception e){

    customMessage.setMessage(e.getMessage());
    customMessage.setDomain(null);
}

```

위에 이런 로직을 추가했다 이름을 판별할때 빈값을 보내거나 이름의 길이가 8보다 작으면 에러를 던지는 것을 개발했는데 
당연히 이렇게 되면 post-man 에서는 에러가 발생한다 

물론 우리는 이런 방식으로 개발을 진행을 했지만 

매 api 마다 동일한 로직이면 동일한 소스를 사용해야 한다 물론 aop 를 활용할 수 있지만 aop 보단 spring vaild 를 활용해보자 


```
public class UserValidation implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return User.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {

        User user = (User)target;

        if(user.getName().length() < 8 || !StringUtils.hasText(user.getName())){
            errors.rejectValue("name" , "name length inValid" , "이름의 길이가 올바르지 않습니다.");
        }

        if(user.getAge() < 10){
            errors.rejectValue("age" , "Invalid age" , "나이가 올바르지 않습니다.");
        }


    }
}


```

기존 도메인 User 에 이와 같은 소스를 추가한다 그리고 validate 에서 이 User 도메인의 검증할 수 있는 분기 로직을 적어준다 
이때 ` errors.rejectValue("name" , "name length inValid" , "이름의 길이가 올바르지 않습니다.");` 파라미터는 순서대로 검증 필드 , 오류 코드 메세지 
오류 코드 같은 경우엔 정해져 있는것이 아니기에 약속 또는 본인이 커스텀한 내용을 적으면 된다 

그럼 서비스 로직도 변경이 되어야 하는데


```

@PostMapping("/")
@ResponseBody
public ResponseEntity<CustomMessage<?>> insertUser(@RequestParam("name") String name , @RequestParam("age") int age)
{

    CustomMessage customMessage = new CustomMessage();

    try{
        User user = new User();

        user.setUserId(users.getUsersList().size());
        user.setName(name);
        user.setAge(age);

        UserValidation userValidation = new UserValidation();
        Errors errors = new BeanPropertyBindingResult(user , "user");

        userValidation.validate(user , errors);

        errors.getAllErrors().forEach(e -> {
            throw new RuntimeException(e.getDefaultMessage());
        });




        users.addUser(user);

        customMessage.setMessage("새로운 유저가 생성되었습니다.");
        customMessage.setDomain(user);

        return new ResponseEntity<>(customMessage , HttpStatus.OK);

    }catch(Exception e){

        customMessage.setMessage(e.getMessage());
        customMessage.setDomain(null);

        return new ResponseEntity<>(customMessage , HttpStatus.INTERNAL_SERVER_ERROR);

    }
}



```
서비스 로직은 기존 로직을 폐기한 후에 

```
UserValidation userValidation = new UserValidation();
Errors errors = new BeanPropertyBindingResult(user , "user");

userValidation.validate(user , errors);

errors.getAllErrors().forEach(e -> {
    throw new RuntimeException(e.getDefaultMessage());
});

```

이 로직을 추가해준다 이때 이 로직은 반드시 새로운 도메인 객체를 생성후 그 객체에 값이 set 된 다음 검증을 유도애햐 한다 그리고 커밋을 해야 하는데 이때 커밋의 역활을 할 코드가 
`users.addUser(user);` 이다 
n
그럼 여기서 궁금한것 `BeanPropertyBindingResult` 이게 뭐냐는 것이다 
공식문서에서는 ErrorsJavaBean 객체에 대한 바인딩 오류의 등록 및 평가를 위한 인터페이스 기본 구현체 즉 Errors 를 사용하기 위해서는 인터페이스 임으로 BeanPropertyBindingResult
객체를 만들어서 구현체로 사용하시면됩니다 첫번째 파라미터는 도메인 타입의 객체 (검증을 할려는) 두번째는 도메인 객체의 이름을 넣어줍니다 

```
userValidation.validate(user , errors);

```
를 통해서 현재 도메인의 문제점을 검증하고 문제가 생기면 errors 에 넣어줍니다 그리고 우리가 그 에러를 꺼낼때에는 

```

errors.getAllErrors().forEach(e -> {
    throw new RuntimeException(e.getDefaultMessage());
});

```
사용해서 던져주면 됩니다 그러면 처음으로 맞이하는 에러를 바로 return 을 시키고 mvc 핸들러를 종료하게 됩니다