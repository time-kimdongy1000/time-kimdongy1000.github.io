---
title: Spring Validator
author: kimdongy1000
date: 2023-03-21 14:20
categories: [Back-end, Spring - Core]
tags: [ Validator ]
math: true
mermaid: true
---

## Validator
이는 스프링 프레임워크에서 제공하는 데이터 유효성 검증을 위한 인테피으스 입니다 사용자로 부터 받은 데이터나 외부 데이터의 유효성을 검사하여 오류를 찾고 
필요한 경우 에러메세지를 만들어냅니다 

## Member 도메인
그러면 예를 들어서 Member 를 만든다고 했을때 이메일 주소와 , 이름과 , 나이를 받는다고 하자 그러면 도메인은 이렇게 만들어질것이다 

```

public class Member {
    
    private String email;
    
    private String username;
    
    private int age;

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}

```

## Member Vaildator 
그러면 각각의 제약조건을 만들어보자 

email 은 필수값이며 반드시 이메일 정규식에 포함이 되어야 한다 
username 필수값이기만 하면된다 
age 는 20 이상 80 이하여야 한다 

이런 조건으로 유효성 검사를 만들게 되면은

```
public class MemberValidator implements Validator {

    
    /**
     * 평범한 이메일 정규식 
     * 
     * */
    private static final String EMAIL_REGEX = "^[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,4}$";
    private static final Pattern pattern = Pattern.compile(EMAIL_REGEX);



    /**
     * 
     * Member class 를 Validator 검증에 사용할 수 있는 클래인지 판별하게 됩니다 
     * 
     * */
    @Override
    public boolean supports(Class<?> clazz) {
        return Member.class.isAssignableFrom(clazz);
    }
    
    /**
     * 
     * 검증 핵심 
     *  email 값이 존재 하냐 안하냐 
     *  email 이 정규식을 통과 할 수 있으냐 없으냐 
     *  username 이 존재하냐
     *  나이가 20 ~ 80 사이냐 를 체크하게 됩니다 
     * 
     * 
     * */

    @Override
    public void validate(Object target, Errors errors) {

        Member member =  (Member) target;
        Matcher matcher = pattern.matcher(member.getEmail());

        if(!StringUtils.hasText(member.getEmail())){
            errors.reject("email" , "이메일은 필수값 입니다");
        }

        if(!StringUtils.hasText(member.getUsername())){
            errors.reject("username" , "username 은 필수값입니다 ");
        }

        if(!matcher.matches()){
            errors.reject("email" , "이메일 형식이 아닙니다");
        }

        if(member.getAge() < 20 || member.getAge() > 80){
            errors.reject("age" , "나이는 20 이상 80 이하여야 합니다 ");
        }

    }
}

```
핵심은 validate 핵심인것입니다 이곳에서 vaild 를 통과하지 못하면 에러를 발생시키는데 그러면 Memeber 를 만들어보겠습니다 


## Member 생성 

```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		try{

			Member member1 = new Member();
			member1.setEmail("time@gmail.com");
			member1.setUsername("time");
			member1.setAge(25);



			MemberValidator memberValidator = new MemberValidator();

            /**
			 * BeanPropertyBindingResult 바인딩 과정에서 생기는 오류를 저장하고 관리하는 클래스입니다 보통 Validation 에서 발생하는 에러를 저장하고 처리합니다 
			 *
			 * */
			Errors errors = new BeanPropertyBindingResult(member1 , "member1");

            /**
            *
            *
            * member1 를 체크하게 됩니다 그리고 그때 발생한 error를 errors 객체에 저장을 하게 됩니다 
            *
            *
            *
            *
            */

			memberValidator.validate(member1 , errors);

            /**
			 * 에러가 발생하게 된다면 if 문을 타고 error 를 하나씩 throws 던지게 됩니다 
			 * 이때는 여러개의 에러가 동시에 발생해도 하나의 에러만 처리하게 됩니다 
			 *
			 * */
			if(errors.hasErrors()){
				errors.getAllErrors().forEach( error -> {
					throw new RuntimeException(error.getDefaultMessage());
				});
			}


		}catch(Exception e){
			e.printStackTrace();
		}
	}
}

```
각각 email 을 뺴거나 유효성을 검증못하게 하거나 등등 다양한 방법으로 바인딩 되기 전에 데이터 검증을 할 수 있습니다 오늘은 스프링의 유효성 검증중에 Validator 에 대해서 알아보았습니다 