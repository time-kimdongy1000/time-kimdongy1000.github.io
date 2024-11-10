---
title: Spring MicroService 22 Spring MicroService 보안2
author: kimdongy1000
date: 2023-08-07 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Security]
math: true
mermaid: true
---

우리는 지난시간에 keyclock 을 사용을 keyclock 이 왜 필요한지 그리고 어떻게 인증이 이루어지는지에 대해서 postman 을 통해서 keyclock 과 연동을 하고 안에 통신을 통해서 정보를 받아오는 것을 진행했다 이번시간에는 직접 지난번 미니프로젝트에 이 부분을 입혀서 어플리케이션 보안을 진행을 해보자 



## 전체소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project-oauth2?ref_type=heads>

## hosts 파일 수정
이번시간부터는 우리가 직접 docker 브릿지 네트워크상에 있는 이름을 이용해서 통신을 진행을 해야 함으로 폴더의 hosts 파일을 수정할것입니다 우리가 직접 이 어플리케이션을 등록해서 네임서버 등록을 못함으로 수동으로 우리가 각각의 브릿지상 네트워크의 네임을 로컬에만 등록해서 사용할것입니다 
`C:\Windows\System32\drivers\etc` 여기 아래에 있는 hosts 파일을 수정하겠습니다 


```

# localhost name resolution is handled within DNS itself.
	127.0.0.1      localhost
	127.0.0.1		eurekaserver
	127.0.0.1		gateway
	127.0.0.1		keycloak
#	::1            localhost

``` 
아마 기본적으로 localhost 는 지정이 되어 있을을 것이고 우리가 docker 상에서 사용하는 컨테이너 이름이 - 네트워크 DNS 가 됨으로 이를 우리가 127.0.0.1 로 매핑을 해주면된다 

## config-server gateway-dev.yml
```
spring:
  cloud:
    gateway:
      discovery.locator:
        enabled: true
        lowerCaseServiceId: true
  main:
    web-application-type: reactive
  security:
    oauth2:
      client:
        registration:
          Keycloak:
            authorization-grant-type: authorization_code
            client-id: spring-cloud-client1-oauth2
            client-name: spring-cloud-client1-oauth2
            client-secret: 5XhSdcd3JygzJ02RHdt5tSeLA6VxbYkx
            redirect-uri: http://gateway:9002/login/oauth2/code/keycloak
            clientAuthenticationMethod: client_secret_post
            scope: openid, email
        provider:
          Keycloak:
            issuer-uri: http://keycloak:8080/realms/spring-cloud-client1-oauth2

```
security 를 추가합니다 만약 저랑 cleint 이름과 realms 가 동일하다면 client-secret 빼고 동일하게 하면됩니다 그렇지 않으면 자신이 설정한 주소를 입력하면되고 그 정보는 
`http://keycloak:8080/realms/[자신이 설정한 client-id]/.well-known/openid-configuration` 여기로 가시면됩니다 이때 주소창자신이 설정한 client-id 를 입력하면 url 정보가 나오게 됩니다 

## gateway pom.xml 
```
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-oauth2-client -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-security -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>

```
gateway 는 의존성 security 와 oauth2 를 사용합니다 그리고 기동을 하고 /createEmp 페이지로 이동을 하겠습니다 이제 우리는 localhost 가 아닌 DNS 로 움직이도록 하겠습니다 

## SecurityWebFilterChain
```
@SpringBootApplication
@EnableEurekaClient
@EnableWebFluxSecurity
public class MiniProjectGateWayApplication {

    public static void main(String[] args) {
        SpringApplication.run(MiniProjectGateWayApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) throws Exception {
        http.csrf().disable()
                .authorizeExchange()
                .anyExchange().authenticated()
                .and()
                .oauth2Login();

        return http.build();
    }



}

```

보통 우리는 기존의 Security 에서는 SecurityFilterChain 으로 bean 을 만들었는데 사실 이 bean 은 동기식 Rest 기반에서만 동작하는 bean 이다 그래서 실제로 기존의 SecurityFilterChain
bean 을정의하면 오류가 나는데 이때는 webFlux 기반인 SecurityWebFilterChain 으로 bean 을 만들어서 설정을 제어해야 합니다 이 설정은 따로 config 파일을 만들어도 되지만 필수로 `@EnableWebFluxSecurity` 어노테이션이나 
Autowired 가 필요합니다 이 부분은 저도 잘 몰라서 처음에는 많이 헤머고 찾다가 보니 이 SecurityWebFilterChain 을 정의해야 하는것을 알았습니다 




## 인증 화면
![1](https://github.com/user-attachments/assets/c6fe1769-e710-43a1-bdcd-ffafaa3e289a)

그러면 우리가 어제 보았던 화면중 일부가 보이게 됩니다 즉 이제 부터 인증 없이는 gateway 를 통할 수 없습니다 우리가 저장한 user 를 입력해서 로그인을 하면 마찬가지로 인증이 진행이 되고 우리가 갈려는 페이지로 이동하는 것을 볼 수 있습니다 이제 security 보안을 한층 챙긴것을 볼 수 있습니다 인증된 사용자만 들어올 수 있게 만들어놓은것입니다 

우리는 이렇게 keyClock 을 이용한 인증을 만들어보았습니다 


