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

## GET (@GetMapping) - 데이터 요청
HttpMethod Get 메서드는 기본적으로 서버의 리소스를 가지고 올때 사용되는 메서드입니다 Spring 에서는 페이지 return 또는 데이터를 가져올 때 사용하는 HTTP 메서드입니다 
이때 MVC 에는 @GetMapping 을 사용해서 핸들러를 만들게 됩니다 

```
@GetMapping("/main/getBoardDetail")
public ResponseEntity<?> getBoardDetail(
        @RequestParam("board_pk") String board_pk
        
        )
{
    return new ResponseEntity<>(new Board_dto(board_pk) , HttpStatus.OK);
}
```
이때 MVC 핸들러는 이런 모양으로 만들어지게 되고 이쪽에 요청을 넣게 되면 이런 모양이 됩니다 


## GET 요청

```
GET /main/getBoardDetail?board_pk=19 HTTP/1.1
Host: localhost:8080

```
이때 요청정보는 이렇게 만들어지게 됩니다 이때 ? 뒤에는 쿼리스트링이라고 해서 api 를 호출할떄 서버에 던지는 요청파라미터 입니다 서버는 board_pk 라는 것을 key 값으로 이를 requestParam 으로 받을 수 있게 됩니다 이때 쿼리 스트링은 여러개를 보낼 수 있습니다 ?key1=value1&key2=value2&key3=value3... 이렇게 여러개의 파라미터를 같이 보낼때에는 이렇게 & 붙여서 여러개를 보낼 수 있습니다만 
주소창의 max 길이만큼만 데이터를 보낼 수 있습니다 

## GET 응답 정보 
```
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 13 Feb 2022 01:20:40 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
    board_pk: 19
    board_title: "[책 리뷰] 너의 이야기"
    board_write: "<p class=\"se-text-paragraph se-text-paragraph-al
    filePath: "/home/kimdongy1000/java/CYBBProject/img/2022_02_06/4102734.jpg"
    filePk: 29
    insert_date: "20220206"
    update_date: null
    user_email: "kimdongy1000@naver.com"
     user_pk: 1
}

```
응답정보를 하나씩 보면 
마찬가지로 시작라인 , 헤더라인 , 공백라인 , 바디라인 이렇게 나눠져있는 모습을 볼 수 있습니다 
1. 시작라인 
    HTTP/1.1 200 이때 200은 HTTP 상태코드를 나타냅니다 (200 정상)
2. 헤더라인 
    Content-Type 부터 공백라인 위까지 (Eonnection) 이때 Content-type 은 다음에 오는 메세지 바디라인의 형식을 나타냅니다 
3. 공백라인 

4. 메세지 바디라인 
    {board_pk : 19 , board_title : "[책리뷰 ] 너의 이야기..."}

이게 HTTP 메서드 중에서 GET 그중에서 데이터 요청에 관한 요청 메세지와 응답 메세지 입니다 


## GET (@GetMapping) - 페이지 요청
데이터 처럼 무엇인가 가져오기 때문에 보통 페이지 요청도 GET 으로 많이 사용을 하게 됩니다 

```
@GetMapping("/main/boardPage")
public String boardPage()
{
    return "boardPage";
}

```
예를 들어서 다음과 같은 핸들러가 있다 이때 

## GET 요청
```
GET /main/boardPage HTTP/1.1
Host: localhost:8080

```

## GET 응답 
```
HTTP/1.1 200
Content-Type: text/html
Transfer-Encoding: chunked
Date: Sun, 13 Feb 2022 01:20:40 GMT
Keep-Alive: timeout=60
Connection: keep-alive

<!DOCTYPE html>
<html>
    <head>
        ...
    </head>
<body>
...
</body>
</html>

```
페이지를 요청할떄는 아까 데이터를 요청하는것과는 다른 응답이 오게 되는데 시작라인은 동일하지만 Content-Type 과 body 라인이 존재하는데 이때 boay 는 html 태그가 넘어오게 되고 이를 크롬 또는 사용하는 클라이언트 프로그램에 알맞게 렌더링을 진행해서 클라이언트에게 보여주게 됩니다 

이처럼 GET 는 데이터를 요청하는 방식과 , 페이지를 요청하는 방식 두가지가 존재합니다 

## POST (@PostMapping) - 데이터 생성 
Post 는 주로 서버에 데이터를 생성하거나 서버의 상태를 변경할려고 할떄 주로 사용하는 HTTP 메서드입니다 

```
@PostMapping("/main/write/writeBoard")
public ResponseEntity<?> getBoardDetail(
        @RequestBody<String , Object> boay_param
        
        )
{
    insertWriteBoard(boay_param);

    return new ResponseEntity<>(null , HttpStatus.OK);
}
```
POST 는 핸들러가 조금 바뀌게 됩니다 클라이언트에서 보낸 파라미터가 쿼리스트링 형태로 오는게 아니라 HTTP 메세지 바디에 담겨서 오는것임으로 이때는 @RequestBody 사용해서 파라미터를 받을 수 있습니다 

## POST 요청
```
POST /main/write/writeBoard HTTP/1.1
Host: localhost:80
Connection: keep-alive
Content-Length: 117

{
    content: "HTTP POST 게시글 작성중입니다"
    fileValue: "30"
    title: "HTTP POST 게시글 "
}

```
마찬가지로 시작라인 , 헤더라인 , 공백라인 , 메세지 바디라인 으로 나눌 수 있습니다 
1. 시작라인 
    POST /main/write/writeBoard HTTP/1.1
2. 헤더라인 
    HOST 공백 라인 앞까지 (Content-Length: 117 까지)    
3. 공백라인

4. 메세지 바디라인  
    {content: "<p>HTTP POST 게시글 작성중입니다" , fileValue: "30" ...}


## post 응답 
```
HTTP/1.1 201 Created
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 13 Feb 2022 01:30:11 GMT
Location: /board/20


```
마찬가지로 응답도 시작라인 , 헤더라인 , 공백라인 , 메세지 바디라인으로 나누게 되는데 
1. 시작라인 
    HTTP/1.1 201 Created 이때 post 는 서버에서 무엇인가를 생성을 했음으로 보통 상태코드 201 과 헤더에 Location 을 보내게 됩니다 
2. 헤더라인
       Content-Type 부터 시작해서 공백라인 위까지 (Location: /board/20 까지) 
3. 공백라인

4. 메세지 바디라인 
    지금 핸들러 자체에서 응답을 null 로 보냈기 때문에 나오지 않습니다 return 을 줄떄 `return new ResponseEntity<>(null , HttpStatus.OK);` null 부분에 특정 값을 넣어주게 되면 HTTP 메세지를 바디라인을 만들어서 응답을 보내게 됩니다 

이 밖에도 PUT , PATCH , DELETE 라는 메서드가 있는데 이에 대해서는 Spring MVC 에서 직접 다루어 보는것으로 하고 HTTP 메서드와 관련된것은 이것으로 마치도록 하겠습니다 