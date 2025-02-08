---
title: Spring MVC i18n
author: kimdongy1000
date: 2023-04-21 10:10
categories: [Back-end, Spring - MVC]
tags: [ MVC , i18n ]
math: true
mermaid: true
---

## i18n
i18n은 국제화 소프트웨어를 다양한 언어와 문화권에서 쉽게 지역화할 수 있도록 준비하는 과정입니다 코드와 데이터의 분리, 문자열 리소스 파일 사용, 유니코드 지원 등 다양한 기술적 준비를 포함합니다.

우리는 이번에 HTTP 메시지를 만들어보고 이들을 국제화 작업을 한번 해보려고 합니다

## StudentController

```
@RestController
@RequestMapping(value ="v1/school/{schoolId}/student")
public class StudentController {

    @Autowired
    private StudentService studentService;

    @GetMapping("/{studentId}")
    public ResponseEntity<Student> getStudent(
            @PathVariable("schoolId") String schoolId ,
            @PathVariable("studentId") String studentId
    )
    {
        return new ResponseEntity<>(studentService.getStudent(studentId , schoolId) , null , HttpStatus.OK);

    }

    @PostMapping
    public ResponseEntity<String> createStudent(
            @PathVariable("schoolId") String schoolId ,
            @RequestBody Student student
    )
    {
        return new ResponseEntity<>(studentService.createStudent(student , schoolId) , null , HttpStatus.OK);
    }

    @PutMapping
    public ResponseEntity<String> updateStudent (
            @PathVariable("schoolId") String schoolId ,
            @RequestBody Student student
        )
    {
        return new ResponseEntity<>(studentService.updateStudent(student,schoolId) , null , HttpStatus.OK);
    }

    @DeleteMapping(value = "/{studentId}")
    public ResponseEntity<String> deleteStudent(
            @PathVariable("schoolId") String schoolId ,
            @PathVariable("studentId") String studentId


    )
    {
      return new ResponseEntity<>(studentService.deleteStudent(studentId , schoolId) , null , HttpStatus.OK);
    }


}


```
학생 정보를 가져오고 입력하고 수정하고 삭제하고 업데이트하는 엔드 포인트입니다 Student POJO 클래스는 다음과 같습니다

## Student
```
@Getter
@Setter
@ToString
public class Student {

    private int id;
    private String studentId;
    private String studentName;
    private String schoolId;
    private String gender;
    private String higherGroup;
}


```

## StudentService

```
@Service
public class StudentService {

    public Student getStudent(String studentId , String schoolId){
        Student student = new Student();
        student.setId(new Random().nextInt(5000));
        student.setStudentId(studentId);
        student.setSchoolId(schoolId);
        student.setStudentName("Kim");
        student.setGender((new Random().nextInt(2) + 1) == 1 ? "Man" : "Woman");
        student.setHigherGroup((new Random().nextInt(2) + 1) == 1 ? "Y" : "N");

        return student;
    }

    public String createStudent(Student student , String schoolId)
    {
        String responseMessage = null;

        if(student != null){
            student.setSchoolId(schoolId);
            responseMessage = String.format("학생입력이 정상적으로 되었습니다 입력된 학생의 정보는 : %s" , student.toString());
        }

        return responseMessage;
    }

    public String updateStudent(Student student , String schoolId)
    {
        String responseMessage = null;
        if(student != null){

            student.setSchoolId(schoolId);
            responseMessage = String.format("학생 업데이트 정상적으로 되었습니다 수정된 학생의 정보는 : %s" , student.toString());

        }

        return responseMessage;

    }

    public String deleteStudent(String studentId , String schoolId)
    {
        String responseMessage = null;
        responseMessage = String.format("학생 번호 %s 는 학교 번호 %s 로 부터 삭제 되었습니다" , studentId , schoolId);
        return responseMessage;

    }


}


```

만약 해당 서비스를 한국어에 국한된 서비스라면 굳이 국제화 작업을 할 필요 없습니다 국제화 작업도 꽤나 노동이 들어가기 때문입니다 하지만 우리는 전 세계를 위한 작업을 해야 하므로 국제화 작업을 진행을 하겠습니다 현재 보이는 controller 와 서비스는 HttpMessage로 한글을 주고 있습니다 이런 식으로 하드코딩을 하게 되면 국제화 작업이 가능하지 않습니다 해당 언어를 위한 서비스를 언어가 추가될 때마다 작업을 해야 하는데 그렇게 할 수 없습니다 그래서 Spring MessageSource를 사용해서 손쉽게 국제화를 진행을 할 수 있습니다

## MessageSource 빈 생성
```
@SpringBootApplication
public class ProjectSpringCloudApplication {

	public static void main(String[] args) {
		SpringApplication.run(ProjectSpringCloudApplication.class, args);
	}
	
	@Bean
	public LocaleResolver localeResolver(){
		SessionLocaleResolver localeResolver = new SessionLocaleResolver();
		localeResolver.setDefaultLocale(Locale.KOREA);
		return localeResolver;
	}
	
	@Bean
	public ResourceBundleMessageSource resourceBundleMessageSource(){
		ResourceBundleMessageSource resourceBundleMessageSource = new ResourceBundleMessageSource();
		resourceBundleMessageSource.setUseCodeAsDefaultMessage(true);
		resourceBundleMessageSource.setDefaultEncoding("UTF-8");
		resourceBundleMessageSource.setBasenames("messages");

		return resourceBundleMessageSource;

	}

}


```

2개의 bean 을 정의하겠습니다 LocaleResolver는 기본으로 사용할 언어의 기본입니다 뒤에 헤더에 특정 나라를 구분하는 코드가 들어오지 않으면 기본적으로 KR 을 제공하겠다는 뜻입니다 ResourceBundleMessageSource는 properties의 어떤 파일을 기본적으로 메시지를 읽는지와 인코딩 설정을 할 수 있습니다

## messageSoruce 만들기

messages.propreteis
```
student.create.message = 학생 입력이 정상적으로 되었습니다 입력된 학생의 정보는 : %s -default_model
student.update.message = 학생 정보수정이 정상적으로 되었습니다 수정된 학생의 정보는 : %s -default_model
student.delete.message = 학생 번호 %s 는 학교 번호 %s 로 부터 삭제 되었습니다 - default_model

```

messages_kr.propreteis
```
student.create.message = 학생 입력이 정상적으로 되었습니다 입력된 학생의 정보는 : %s -kr_model
student.update.message = 학생 정보수정이 정상적으로 되었습니다 수정된 학생의 정보는 : %s -kr_model
student.delete.message = 학생 번호 %s 는 학교 번호 %s 로 부터 삭제 되었습니다 - kr_model

```

messages_en.propreteis
```
student.create.message = Student input has been successfully received. The entered student information is: %s -en_model
student.update.message = Student information has been successfully updated. The updated student information is : %s -en_model
student.delete.message = StudentId %s and SchoolId %s was Deleted.- en_model

```

총 3가지 메시지 정보를 만들었다 기본적으로 특정 헤더에 국가 코드가 들어오지 않으면 message.propreteis 정보가 들어올 것이고 특정 국가 코드를 넣게 되면 해당 국제화 표준모델로 메시지가 나가게 됩니다 그럼 Controller 과 Serivce를 수정해야 합니다 이때 각 꼬리표는 확인을 위함입니다 국제화 모델하고는 상관이 없습니다

## Controller 수정

```

 @PostMapping
    public ResponseEntity<String> createStudent(
            @PathVariable("schoolId") String schoolId ,
            @RequestBody Student student,
            @RequestHeader(value = "Accept-Language" , required = false) Locale locale
    )
    {
        return new ResponseEntity<>(studentService.createStudent(student , schoolId , locale) , null , HttpStatus.OK);
    }

    @PutMapping
    public ResponseEntity<String> updateStudent (
            @PathVariable("schoolId") String schoolId ,
            @RequestBody Student student,
            @RequestHeader(value = "Accept-Language" , required = false) Locale locale
        )
    {
        return new ResponseEntity<>(studentService.updateStudent(student,schoolId ,locale) , null , HttpStatus.OK);
    }

    @DeleteMapping(value = "/{studentId}")
    public ResponseEntity<String> deleteStudent(
            @PathVariable("schoolId") String schoolId ,
            @PathVariable("studentId") String studentId ,
            @RequestHeader(value = "Accept-Language" , required = false) Locale locale


    )
    {
      return new ResponseEntity<>(studentService.deleteStudent(studentId , schoolId , locale) , null , HttpStatus.OK);
    }


```

3개의 핸들러를 수정할 것입니다 이때 RequestHeader에서 특정 국가 코드를 Accept-Language key 값으로 받아서 locale에 넣어줄 것입니다 그것을 service로 넘겨서 특정 국가 코드의 국제화 모듈을 실행합니다

## Service 수정
```
    @Autowired
    private MessageSource messageSource;

   public String createStudent(Student student , String schoolId , Locale locale)
    {
        String responseMessage = null;

        if(student != null){
            student.setSchoolId(schoolId);
            responseMessage = String.format(messageSource.getMessage("student.create.message" , null , locale) , student.toString());
        }

        return responseMessage;
    }

    public String updateStudent(Student student , String schoolId , Locale locale)
    {
        String responseMessage = null;
        if(student != null){

            student.setSchoolId(schoolId);
            responseMessage = String.format(messageSource.getMessage("student.update.message" , null , locale) , student.toString());

        }

        return responseMessage;

    }

    public String deleteStudent(String studentId , String schoolId , Locale locale)
    {
        String responseMessage = null;
        responseMessage = String.format(messageSource.getMessage("student.delete.message" , null , locale) , studentId , schoolId);
        return responseMessage;

    }

```
3개의 서비스 핸들러를 수정했습니다 MessageSource 의존성을 주입 후 해당 의존성으로 properties에 매핑된 각 메시지 정보를 가져오는데 이때 가져오는 정보는 locale에 담겨 있습니다

## HTTP 요청정보 만들기 
```
POST /v1/school/1/student HTTP/1.1
Host: localhost:8080
Accept-Language: en
Content-Type: application/json
Content-Length: 162

{
    "id": 600 , 
    "studentId" : "2366",
    "studentName" : "MI KO",
    "schoolId" : "1" , 
    "gender" : "woman",
    "higherGroup" : "Y"
    

}

```
만약 이와 같이 Accept-Language: en으로 헤더를 날려주면 MessageSource는 국제화 모듈에 맞춘 messages_en.properties의 값을 가져오게 됩니다

```
Student input has been successfully received. The entered student information is: Student(id=600, studentId=2366, studentName=MI KO, schoolId=1, gender=woman, higherGroup=Y) -en_model
```

```
PUT /v1/school/1/student HTTP/1.1
Host: localhost:8080
Accept-Language: kr
Content-Type: application/json
Content-Length: 162

{
    "id": 600 , 
    "studentId" : "2366",
    "studentName" : "MI KO",
    "schoolId" : "2" , 
    "gender" : "woman",
    "higherGroup" : "Y"
    

}

학생 정보 수정이 정상적으로 되었습니다 수정된 학생의 정보는 : Student(id=600, studentId=2366, studentName=MI KO, schoolId=1, gender=woman, higherGroup=Y) -kr_model

```

kr 요청도 이와 같이 들어오게 되고 마지막 국제화를 만들어놓지 않은 경우는 default.messagesProperties를 향하게 됩니다

```
DELETE /v1/school/1/student/20?Accept-Language=UK HTTP/1.1
Host: localhost:8080

학생 번호 20 는 학교 번호 1 로 부터 삭제 되었습니다 - default_model

```
실제 UK 모델은 아직 만들지 않았기 때문에 지금처럼 default_model 출력이 됩니다 오늘은 하나의 애플리케이션에서 여러 국제화 모듈을 사용하는 i18n에 대해서 알아보았습니다



