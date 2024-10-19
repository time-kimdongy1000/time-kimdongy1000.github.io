---
title: Spring MicroService 26 Spring MicroService 보안
author: kimdongy1000
date: 2023-08-06 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  GateWay]
math: true
mermaid: true
---

우리는 지난 시간까지 Spring-cloug-Gateway를 공부하면서 MSA와 관련된 모든 하위 인스턴스 접근은 GateWay로 통한다는 것을 배웠고 이를 통해서 각 하위 인스턴스와 통신 및 Gateway의 사전 필터 사후 필터를 만들어서 API 추적까지 진행을 해보았습니다

이번 시간에는 MSA의 보안에 대해서 배울 것인데 여러 보안이 있겠지만 우리는 애플리케이션 계층에 대한 보안을 공부할 것입니다

keyclock는 우리가 앞에서 oauth2 공부할 때 한번 해보았으니 자세한 설명은 생략하겠습니다

## 애플리케이션 보안 
적절하게 사용자를 통제하여 사용자 본인 여부와 수행하려는 작업의 수행 권한이 있는지 확인

## 사용할려는 툴
아마 스프링 시큐리티 중 Oauth2 파트를 공부해 본 사람이라면 익숙한 애플리케이션입니다 이는 서드파티 애플리케이션으로 서비스와 애플리케이션을 위한 ID 및 액세스 관리용 오픈소스 설루션입니다
그리고 직접 만든 JWT를 생성하여 인증/인가를 하는 방법도 공부를 해볼 예정입니다

## 도커에 KEYCLOCK 설치
```

 Keycloak:
    image: quay.io/keycloak/keycloak:26.0.1
    restart: always
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME : admin
      KC_BOOTSTRAP_ADMIN_PASSWORD : admin
    ports:
       - 9090:8080
    command: start-dev        
    container_name: Keycloak

```

도커에 keyclock 은 compose 파일에 같이 넣어두도록 하겠습니다 버전은 사용하고 싶은 버전이 있으면 26.0.1 이렇게 명시를 하시거나 아니면 latest 을 사용하시면 됩니다
지금 글을 적는 버전으로는 26.0.1 이 최신 버전입니다

## KEYCLOCK 설정
우리는 앞으로 이 keyclock를 사용하기 위해서 약간의 세팅을 하겠습니다


## 초기화면

![1](https://github.com/user-attachments/assets/90ed5377-9b41-422f-a732-5a98a60eea98)

기동을 하고 설정한 PORT로 들어가면 다음과 같은 화면이 나오는데 이때 설정한 아이디 비밀번호 admin , admin으로 접속을 합니다

## Realm 만들기

![2](https://github.com/user-attachments/assets/5ca828af-8ab2-4326-9400-0489c4a73227)

참고로 realm 은 영어 단어로 왕국이라는 뜻입니다 우리가 사용할 왕국을 만들어볼까요? 좌측 파란색 버튼을 클릭합니다 

![3](https://github.com/user-attachments/assets/9cb6f549-37d9-454b-b4b5-84f54cb44e47)

그리고 이때 이름은 spring-cloud-client1-oauth2로 만들겠습니다

## Client 설정

![4](https://github.com/user-attachments/assets/f7632862-98d4-4418-ac60-0de677d56f5e)

왕국을 만들었으니 왕을 만들어보겠습니다 상단에 있는 create client를 클릭합니다

![5](https://github.com/user-attachments/assets/ec672ac3-3848-44fc-9c74-a377c6def8c3)

마찬가지로 이름은 spring-cloud-client1-oauth2

![6](https://github.com/user-attachments/assets/0d3c96f2-abac-442d-8df7-cc94d68313a1)

다음 세팅은 이 와 같이 만들어줍니다 

![7](https://github.com/user-attachments/assets/c82784aa-5a9a-489c-b23e-d62af11634c8)

여기까지는 따라가고 난중 설정은 바꿀 수 있습니다

![8](https://github.com/user-attachments/assets/286524f6-e818-4a5c-a636-426cf40148cc)

그리고 만들어진 client를 클릭합니다 제일 최하단 spring-cloud-client1-oauth2를 클릭

## Roles 설정

![9](https://github.com/user-attachments/assets/3d1f20cf-49bd-483c-83ab-fb845ccb1cb2)

다음은 Role 을 설정을 할 것입니다 상단에 Create role를 클릭합니다

![10](https://github.com/user-attachments/assets/c0359208-2b93-482c-bd0c-983b2143d0ba)

그리고 우리는 2개의 Role 을 만들 것인데 2개의 Role 은 Master_Role , User1_Role로 만들어줍니다

## Users 추가

![11](https://github.com/user-attachments/assets/c9423e25-0bef-484c-b895-7fb597ecd4d8)

이제 우리가 Users를 추가할 것입니다 상단에 Add user를 클릭합니다

![12](https://github.com/user-attachments/assets/51f7d473-1b5d-42f8-978e-d22ce2eb7991)

이때 우리는 2명의 User를 만들 것이고 Master_User 와 User1_User를 만들어줍니다

![13](https://github.com/user-attachments/assets/d28c31da-6b6e-4ac3-bd55-a57ab9c4b4f4)

그리고 각각에 만들어진 user는 비밀번호를 세팅합니다

## User 에 Role 추가

![14](https://github.com/user-attachments/assets/1079cda3-f990-4401-b971-55ff815761cf)

우리는 앞에서 만든 Role 을 매핑할 것입니다 매핑하는 순서는 Master_User에는 Master_Role를 매핑하고 User1_User User1_Role 매핑해줍니다 이때 매핑 방법은 상단에
Assign role를 클릭해서 방금 만들어준 Role 을 매핑해주면 됩니다

## scope 추가
![15](https://github.com/user-attachments/assets/cf0398a7-c12d-4d8f-8822-d6e2facc099c)

그리고 사용할 scope를 사용할 것입니다 이때 openid로 세팅을 하고 만들 때에는 Create client scope를 통해 만들어주시면 됩니다


## Oauth2 로그인 시작 
우리는 설정은 다 했으므로 Oauth2 인증으로 Authorization_Grant_type으로 인증을 해보도록 하겠습니다

## 승인코드 받기 

![16](https://github.com/user-attachments/assets/ee5ec977-6892-481d-90d1-b2aca8db89dd)

아래의 주소를 인터넷 창에 입력하면 다음과 같은 로그인 창이 나오게 됩니다 이때 우리가 만든 유저 한 명으로 로그인을 하겠습니다 주소는 아래 적어두었습니다

```
http://localhost:9090/realms/spring-cloud-client1-oauth2/protocol/openid-connect/auth?client_id=spring-cloud-client1-oauth2&response_type=code&scope=openid&redirect_uri=http://localhost:8081

```

## 액세스 코드 받기 

![17](https://github.com/user-attachments/assets/ca7b023e-d5bd-4970-b958-d6ac600fbfa8)

그리고 로그인을 하게 되면 다음과 같은 주소가 나오게 됩니다 

## 코드 결과값 추출
```

http://localhost:8081/?session_state=45074d3c-55cf-4f65-acbb-7cbf9174d5ba&iss=http%3A%2F%2Flocalhost%3A9090%2Frealms%2Fspring-cloud-client1-oauth2&code=5dc053d4-baff-43b2-a638-b8c73081f771.45074d3c-55cf-4f65-acbb-7cbf9174d5ba.4ed0d047-e935-4ecf-9fd1-079621d520b6


```
이때 페이지는 404에러가 날것이지만 이때 code라는 파라미터 값을 가져와서 아래에서 postman 을 호출합니다


## postman request 값
```

POST /realms/spring-cloud-client1-oauth2/protocol/openid-connect/token HTTP/1.1
Host: localhost:9090
Content-Type: application/x-www-form-urlencoded
Content-Length: 260

grant_type=authorization_code&code=6a51807f-97f9-4c6c-823a-748a901f74aa.6a33bffa-44cc-49f5-9eed-29e8626700b6.4ed0d047-e935-4ecf-9fd1-079621d520b6&redirect_uri=localhost%3A8081&client_id=spring-cloud-client1-oauth2&client_secret=bABZmHUrA02xSNzNav1mVPUlhs0XANA6

```

## client_secret 값

![18](https://github.com/user-attachments/assets/7f287919-b948-4677-af03-15d48c9ef33a)

참고로 client_secret 값은 다음의 페이지에서 가져올 수 있습니다 개개인의 모든 값이 다르므로 자신이 생성한 값을 가져와야 합니다 

## userinfo 정보 가져오기

![19](https://github.com/user-attachments/assets/7a01240f-1887-4708-981a-cefe73a5f08d)

GET /realms/spring-cloud-client1-oauth2/protocol/openid-connect/userinfo?scope=openid&redirect_uri=http://localhost:8081 HTTP/1.1
Host: localhost:9090

그리고 마지막으로 access_token 을 가져와서 다음의 요청을 넣고 유저 정보를 가져오면 우리가 하려고 하는 Authorization_Grant_type으로 인증을 완료했습니다








