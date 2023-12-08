---
title: Spring MVC HttpMethod
author: kimdongy1000
date: 2023-04-13 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC ,HttpMethod ]
math: true
mermaid: true
---

## HttpMethod
HTTPMethod 란 HTTP 프로토콜에서 사용되는 HTTP 메서드를 나타내는 표현방식입니다 
HTTPMethod 는 클라이언트가 서버에 요청을 보낼 떄 어떤 동작을 수행할 것인지를 정의하며 대표적으로는 GET , POST , PUT , Patch , DELETE 가 있습니다 

## HttpMethod - Spring mvc 
우리가 앞에서 정의할때 기본적으로 핸들러 상단에 정의를 하고 시작합니다 
@GetMapping , @PostMapping 등으로 정의를 하고 클라이언트가 보내는 method 종류하고 , 서버 핸들러가 받는 method 가 일치해야 하며 일치하지 않으면 Spring mvc 는 405 에러가 발생합니다 

우리는 하나하나씩 알아보며 각 메서드에 맞게끔 핸들러를 작성해보겠습니다 

## maven 세팅

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

```

기본적으로 web을 이용할것이고 간단한 CRUD 로 유저를 생산하는것을 method 로 표현할것이라서 db를 직접 연결하기 보다는 jpa 와 h2 를 활용해서 경량으로 간단한 애플리케이션을 만들어볼 예정입니다 


application.properties
```

spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html

```
그리고 타임리프 웹 랜더링 사용하기 위해서 그러면 기초설정은 끝이고 이제 하나씩 살펴보자 

## GET 
HttpMethod Get 메서드는 기본적으로 서버의 리소스를 가지고 올때 사용되는 메서드입니다 우리가 가장 대중적으로 사용하는곳은 바로 html url 렌더링 명시하기할때 입니다 

```

@GetMapping("/hello")
public String hello(){
    
    return "hello";
}

```
서버에는 @GetMapping 핸들러로 받을 수 있습니다 다른 방식으로는 

```

@RequestMapping(value = "/hello" , method = RequestMethod.GET)
public String hello(){

    return "hello";
}

```

이렇게 @RequestMapping 를 활용해서 사용할 수 있습니다 @RequestMapping(value = "/hello" , method = RequestMethod.GET) == @GetMapping("/hello") 임으로 편하신것을 사용하시면되지만 저는 주로 @GetMapping 를 활용하는 편입니다 

그리고 리소스는 단순 웹페이지 뿐만아니라 현재 이 어플리케이션에서 조회할 수 있는 모든 자원들을 뜻합니다 예를 들어서 특정 유저를 가져오는 보여주는것도 만들어보겠습니다 
JPA 를 사용함으로 JPA 코드가 나오지만 이에 대해서는 기술하지 않습니다 



```
@Controller
public class HelloController {

    @Autowired
    private UserService userService;


    @GetMapping("/hello")
    public String hello(){

        userService.insertTempUser();
        
        return "hello";
    }
}


```
우리가 처음 hello 요청을 할떄 임시유저를 하나 만들겠습니다 사실 만드는것은 post 영역인데 이에 대해서는 아래에 다시 기술하겠습니다 


UserService.java
```
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public void insertTempUser() {

        UserEntity tempUser = new UserEntity("Time" , 30);

        Long longId = userRepository.save(tempUser).getId();
        System.out.println("=====================");
        System.out.println("longId" + longId);
        System.out.println("=====================");
        userRepository.flush();
    }
}
서비스 로직 호출하면서 임시 유저를 생성해서 넣겠습니다 


```

UserRepository.java
```
@Repository
public interface UserRepository extends JpaRepository<UserEntity , Long> {


}

```


UserEntity.java
```
@Entity
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;

    private int age;

    public UserEntity() {

    }

    public UserEntity(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
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

UserDto.java
```
public class UserDto {

    private Long id;
    private String name;
    private int age;

    public UserDto(Long id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
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
여기서 잠깐 설명을 하자면 JPA 는 이제 mybatis 에서 xml 역활을 대신하게 됩니다 기본적으로 CRUD 가 있는상태이고 이는 직접 명시하지 않아도 부모에 있는메서드를 가져다 쓰는 상속구조이기 때문에 UserRepository 는 비어 있어도 됩니다 

그리고 UserEntity 는 테이블 형태로 DB 에서 테이블명을 뜻하며 이 java 클래스는 테이블을 뜻하게 됩니다 그리고 UserDto 는 UserEntity 데이터를 인계받아서 UserDto 로 인계하는 역활을 합니다 

```

@GetMapping("/hello2TempUser")
public ResponseEntity<UserDto> getUser(@RequestParam String id)
{

    try{
        Long longId = Long.parseLong(id);
        UserDto returnUser = userService.getUser(longId);

        return new ResponseEntity<>(returnUser , HttpStatus.OK);

    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

```
그리고 @GetMapping("/hello2TempUser") 는 id 파라미터를 받아서 UserEntity 테이블을 뒤져서 리소스를 반환하는 핸들러 입니다 이렇게 우리는 HttpMethod Get 의 역활은 
애플리케이션 내의 리소스를 가져와서 반환하는것을 보았습니다.

## POST
POST 메서드는 서버에 데이터를 제줓하기 위한 HTTP 메서드입니다 이때는 새로운 리소스 생성하거나 업데이트 할때 사용되지만 Restful 기반으로는 Put , Patch 로만 업데이트하고 
POST 는 주로 리소스 생성에 주로 이용됩니다 우리는 그럼 위에서 임시 유저 생성을 없애고 name 하고 age 를 입력할때 새로운 user 를 생성하는 api 를 만들어보면 

```

@PostMapping("/helloCreateUser")
public ResponseEntity<Long> createUser(@RequestParam String name , @RequestParam int age)
{
    try{

        UserDto userDto = new UserDto(name , age);
        Long returnId = userService.createUser(userDto);

        return new ResponseEntity<>(returnId , HttpStatus.OK);

    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

```
지금보이는것이 새로운 핸들러입니다 이 핸들러는 name , age 를 파라미터로 던져주면 이때 새로운 리소스를 생성하는것이 Post 메서드의 큰 특징입니다
그래서 위 핸들러를 조금 수정해서 PRG 요청으로 변경하면(PostRedirectGet) 으로 변경하면 다음처럼 변경될것이다 

```

@GetMapping("/hello2TempUser")
public ResponseEntity<UserDto> getUser(@RequestParam(required = false) String id , Model model)
{

    try{
        Long longId = id == null ? (Long)model.asMap().get("id") : Long.parseLong(id);
        UserDto returnUser = userService.getUser(longId);

        return new ResponseEntity<>(returnUser , HttpStatus.OK);

    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

@PostMapping("/helloCreateUser")
public String createUser(@RequestParam String name , @RequestParam int age , RedirectAttributes redirectAttributes)
{
    try{

        UserDto userDto = new UserDto(name , age);
        Long returnId = userService.createUser(userDto);

        redirectAttributes.addFlashAttribute("id" , returnId);

        return "redirect:/hello2TempUser";

    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

```

이런식으로 변경될것이다 이게 PRG 형태이며 PostRedirectGet 요청이 되는것이다 다시 한번 보자면 Post 메서드는 새로운 리소스를 생성할때 가장 많이 쓰이는 메서드이다 

## Put 
서버에 데이터를 업데이트 하기 위한 HTTP 요청메서드입니다 일반적으로 Put 메서드는 클라이언트 리소스 전체 내용을 서버에 전달하고 그 내용을 전체적으로 수정하는 메서드입니다 
이때는 클라이언트에서 보내는 정보와 , 저장되는 리소스가 동일한 결과를 가지게 됩니다 


```

@PutMapping("/helloPutUser")
public String putUser(@RequestBody UserDto userDto , RedirectAttributes redirectAttributes)
{
    try{

        Long longId = userService.putUser(userDto);

        redirectAttributes.addFlashAttribute("id" , longId);

        return "redirect:/hello2TempUser";

    }catch(Exception e){
        throw new RuntimeException(e);
    }


}

```
핸들러는 이렇게 @PutMapping 로 시작하고 안에 @RequestBody 로 데이터를 받습니다 이때 중요한것은 전체 데이터를 덮어쓰는 기준으로 PutMethod 를 사용하기 때문에 
클라이언트에도 전체데이터를 보내주어야 합니다 

```

   public Long putUser(UserDto userDto)
    {
        Optional<UserEntity> opUserEntity = userRepository.findById(userDto.getId());
        if(opUserEntity.isPresent()){

            UserEntity userEntity = new UserEntity();
            userEntity.setId(userDto.getId());
            userEntity.setAge(userDto.getAge());
            userEntity.setName(userDto.getName());


            return userRepository.save(userEntity).getId();


        }else{
            throw new RuntimeException("데이터가 존재하지 않습니다");
        }

        
    }

```
그리고 업데이트 하는 과정은 findbyId 해서 엔티티에서 User 정보를 찾은 다음에 opUserEntity.isPresent() 찾은 유저가 있으면 전체데이터를 업데이트 그것이 아니면 없는 데이터 임으로 
throws 또는 insert 를 하는 로직을 만들 수 있습니다 

## Patch 
 Patch 는 Put 과 마찬가지로 데이터를 업데이트하는 성격은 같지만 Put는 전체데이터를 덮는 과정이라면 Pacth 는 데이터 일부분만 수정하떄 사용할 수 있습니다 

 ```

@PatchMapping("/helloPatchUser")
public String patchUser(@RequestParam Long id , @RequestParam String name ,  RedirectAttributes redirectAttributes)
{
    try{
        
        Long longId = userService.patchUser(id , name);

        redirectAttributes.addFlashAttribute("id" , longId);

        return "redirect:/hello2TempUser";
        
    }catch(Exception e){
        throw new RuntimeException(e);
    }
    
}

 ```
 핸들러는 이렇게 작성이 될것이다 이 핸들러는 User엔티티의 key 값을 받아서 유저를 식별하고 이름을 변경하는 Patch Method 이다 마찬가지로 변경된 뒤에는 Id로 변경된 
 리소스를 다시 Get 하는 redirect 로 이루어져 있다 

 ## Delete 
서버에서 저장된 리소스를 삭제하기 위해 전달되며 이때 body 에 내용을 포함하며 삭제 여부만 전달되게 됩니다 

```

@DeleteMapping("/deleteUser/{id}")
public ResponseEntity<String> deleteUser(@PathVariable Long id)
{
    try{
        userService.deleteUser(id);

        return new ResponseEntity<>( "유저가 삭제되었습니다." , HttpStatus.OK);
    }catch(Exception e){
        throw new RuntimeException(e);
    }
}


```

```

public void deleteUser(Long id) {

    Optional<UserEntity> opUserEntity = userRepository.findById(id);

    if(opUserEntity.isPresent()){
        userRepository.delete(opUserEntity.get());
    }else{
        throw new RuntimeException("데이터가 존재하지 않습니다");
    }


}


```
이렇게 만들 수 있습니다 그리고 한번더 같은 id 로 조회 하면 없는 값이라고 나오게 됩니다 

이렇게 우리는 HttpMethod 로 여러가지 핸들러를 직접 만들어보았습니다