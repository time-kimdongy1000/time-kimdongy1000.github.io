---
title: Spring MVC HATEOAS
author: kimdongy1000
date: 2023-04-21 10:20
categories: [Back-end, Spring - MVC]
tags: [ MVC , HATEOAS ]
math: true
mermaid: true
---

## HATEOAS 
HATEOAS 해당 리소스와 관련된 링크를 표시하는것을 말합니다 이 원칙에 따르면 API 는 각 서비스 응답과 함께 가능한 다음단계 정보도 제공하며 클라이언트를 다음 단계로 가이드 할 수 있어야 한다 

그럼 지난시간에 했던 Student 로 간단하게 HATEOAS 를 만들어보겠습니다 

## 의존성 추가
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>


```
이 의존성이 HATEOAS 와 관련된 의존성입니다 

## Student 모델 수정
```
@Getter
@Setter
@ToString
public class Student extends RepresentationModel<Student> {

    private int id;
    private String studentId;
    private String studentName;
    private String schoolId;
    private String gender;
    private String higherGroup;
}


```
우리가 원래 사용하던 model 에서 RepresentationModel 를 상속받아 줍니다 

## GET 요청시 HATEOAS 생성해서 body 에 전달 

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
        Student student = studentService.getStudent(studentId , schoolId);

        Link selfLink = WebMvcLinkBuilder.linkTo(WebMvcLinkBuilder.methodOn(StudentController.class).getStudent(schoolId , studentId)).withSelfRel();
        Link createLink = WebMvcLinkBuilder.linkTo(WebMvcLinkBuilder.methodOn(StudentController.class).createStudent(schoolId , student , null)).withRel("createStudent");
        Link updateLink = WebMvcLinkBuilder.linkTo(WebMvcLinkBuilder.methodOn(StudentController.class).updateStudent(schoolId , student , null)).withRel("updateStudent");
        Link deleteLink = WebMvcLinkBuilder.linkTo(WebMvcLinkBuilder.methodOn(StudentController.class).deleteStudent(studentId , studentId , null)).withRel("deleteStudent");
        student.add(selfLink , createLink , updateLink , deleteLink);
        return new ResponseEntity<>(student, null , HttpStatus.OK);

    }

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


}


```
지금 보면 GET 요청에 현재 사용하고 있는 메서드를 만들어서 return 하고 있는 모습을 볼 수 있습니다 

```
/**

    이 링크는 자기 자신을 부를때 사용하는 링크라서 마지막에 withSelfRel 을 두는것이고 

*/
Link selfLink = WebMvcLinkBuilder.linkTo(WebMvcLinkBuilder.methodOn(StudentController.class).getStudent(schoolId , studentId)).withSelfRel();

/**

    이 링크는 자기자신 을 부르는것이 아니기 때문에 withRel 을 사용해서 메서드 명을 지정해줍니다

*/
Link createLink = WebMvcLinkBuilder.linkTo(WebMvcLinkBuilder.methodOn(StudentController.class).createStudent(schoolId , student , null)).withRel("createStudent");



```
전체적으로 보면 StudentController 안에 핸들러에 각 메서드를 체이닝 걸고 그때 필요한 파라미터를 각각 넣어주면 됩니다 그리고 자기 자신의 핸들러에서는 withSelfRel 로 link 를 만들고 
그렇지 않은 링크들은 withRel 과 메서드 명을 적어서 끝을 냅니다 이렇게 하고 기동을 하고 post 맨으로 get 요청을 시도하면

## postMan 결과 
```
{
    "id": 4768,
    "studentId": "2205",
    "studentName": "Kim",
    "schoolId": "1",
    "gender": "Woman",
    "higherGroup": "N",
    "_links": {
        "self": {
            "href": "http://localhost:8080/v1/school/1/student/2205"
        },
        "createStudent": {
            "href": "http://localhost:8080/v1/school/1/student"
        },
        "updateStudent": {
            "href": "http://localhost:8080/v1/school/1/student"
        },
        "deleteStudent": {
            "href": "http://localhost:8080/v1/school/2205/student/2205"
        }
    }
}

```
이렇게 이렇게 클라이언트에게 다음의 액션을 취할 수 있게 링크를 제공합니다 

정리하자면 클라이언트가 해당 웹사이트에서 사용할 수 있는 링크들을 탐색하고 싶을때 서버측에서 제공하는 서비스 기능중 하나입니다 클라이언트는 서버가 제공해주는 것을 보고 
다음 스텝을 진행을 합니다 