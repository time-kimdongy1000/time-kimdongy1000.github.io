---
title: Spring MVC UrlHandler
author: kimdongy1000
date: 2023-04-12 15:39
categories: [Back-end, Spring - MVC]
tags: [ MVC , PathVariable , RequestParam , RequestBody ]
math: true
mermaid: true
---

## Spring MVC - UrlHander
우리는 앞에서 ModelAttirbute 로 클라이언트에서 던진 파라미터를 받아서 세팅하는 것을 보았다 이번엔 다양한 방식으로 클라이언트에서 던지는 파라미터를 받아오는 방식에 대해서 
공부를 해볼것이다 

## @PahtVariable 
이 요청방식은 우리가 요즘 흔히보는 url 모양이다 대표적으로 네이버 블로그 url 주소를 한번 살펴보자
https://blog.naver.com/kimdongy1000/221831687013 
이런 형태이다 이런 형태는 프로토콜://도메인/naver아이디/게시글번호 전형적인 PahtVariable 로 구현이 되어 있는 url 모양이다 그럼


HelloController.java
```
@RestController
public class HelloController {


    @GetMapping("/time/{boardNumber}")
    public String path(@PathVariable String boardNumber){
        
        return boardNumber;

    }

}


```
핸들러는 간단하게 만들수 있다 클라이언트에서 변수로 던저질 부분에 {key} 로 감싸면된다 그리고 그러면 우리는 클라이언트 주소 요청은 이렇게 될것이다 
http://localhost:8080/time/2223311 우리는 요청을 이렇게 넣을것이고 이때 클라이언트의 변수값은 2223311 게 될 수 있는것이다 
그리고 @PathVariable 앞에 key 값을 매핑을 시킬 수 있는데 

```

@GetMapping("/id/{id_key}/board/{boardNumber}")
public String path(@PathVariable("boardNumber") String boardNumber , @PathVariable("id_key") String id_key){

    return id_key + boardNumber;

}


```

이렇게 사용할 수 있게 된다 즉 여러개를 쓸때는 key 값으로 구분을 해주는 좋다 그게 아니라도 key 값을 적지 않으면 변수 이름으로 알아서 매핑이 된다 
물론 요청하는 식은 http://localhost:8080/id/time/board/2223311  이렇게 되는것이다

당연히 Post 로도 동작한다 물론 Post 로 동작하면 url 웹이 아닌 postman 같은 클라이언트 서드파티 어플리케이션을 사용해야 한다 


## @RequestParam
Rest개발 이전에 가장많이 사용하는 클라이언트에서 파라미터 전달방법책이다 
key - value 로 이루어진 url 주소를 파싱해서 데이터를 세팅하세 되는데 핸들러 모양은 이렇게 될것이다 

```

@GetMapping("/id/board")
public String path2(@RequestParam String id , @RequestParam String boardNumber){
    
    
    return id + boardNumber;
    
}


```

핸들러의 모양은 이렇게 된다 그럼 이 주소는 우리가 예측하기로 http://localhost:8080/id/board?id=time&boardNumber=213123 이런모양으로 요청을 넣어야 한다 
postman 으로 바로 호출하면되지만 우리는 form - controller 로도 만들어보자 

```
<!DOCTYPE html>
<html lang="en">
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project 2


<form id="form-table" action="/id/board" method="get">
    
    id : <input name = "id">
    boardNumber : <input name = "boardNumber">


    submit : <input class = "form-button" type="submit" value = "제출">

</form>



</body>
</html>

```
타임리프 무시하고 그냥 간단한 form - controller 만들었다 역시나 주소창에 http://localhost:8080/id/board?id=time&boardNumber=123123 노출이 되면서 다음 페이지로 넘어간다 


그럼 이를 post 로 작성을 해보자 기본적으로 post 작성은 파라미터를 body 에 담아서 보내는것이 특징이다 
먼저 post 핸들러를 작성하자 

```

@PostMapping("/id/board")
public String path3 (@RequestParam String id , @RequestParam String boardNumber){

    return id + boardNumber;
}

```
핸들러는 이렇게 간단하게 그냥 작성이 된다 그저 GetMapping -> PostMapping 로 작성을 하면된다 
그런 form-controller 은 어떻게 작성이 되느냐?

```

<form id="form-table" action="/id/board" method="post">

    id : <input name = "id">
    boardNumber : <input name = "boardNumber">


    submit : <input class = "form-button" type="submit" value = "제출">

</form>


```
기존의 메서드 get 에서 post 로 변경을 하면된다 post 요청의 특징은 안에 값을 넣고 돌려도 http://localhost:8080/id/board 주소창에는 파라미터가 노출이 되지 않는다 그래서 
민감정보같은 경우는 get 요청에서 url 노출보다는 post 요청을 통해서 데이터를 요청body 에 담아서 보내는것이 좋다 이때 요청정보 body 를 requestBody 라고 하는데 이에 대해서는 밑에서 알아볼 예정이고 이를 postman 으로 작성을 해보자 

![1](https://user-images.githubusercontent.com/58513678/231376687-acb46be2-af5d-4957-8862-7ab6563c5d82.jpg)

이와 같이 주소는 기본주소를 써놓고 body 부분의 form-url-encoder 에 key - value 를 작성해서 적어주는것이다 
그리고 요청을 하게 되면 들어오게 된다 

## @RequestBody 
마지막으로 이에 대해서 알아볼것이다 이는 우리가 앞에서 post 작성방법에서 잠깐 보긴 했지만 실제로는 이렇게 활용된다 먼저 핸들러를 작성해보자 

```

@PostMapping("/id2/board")
public String path5 (@RequestBody Map<String , Object> requestBody){

    return requestBody.get("id").toString() + requestBody.get("boardNumber").toString();
}

```
기본적으로 body 로 던져지는 값들은 key - value 형태로 넘어가게 되며 이를 핸들러에서 이와 같이 Map 형태로 받을 수 있다 그럼 이를 안에서 map 형태로 꺼낼 수 있다는 것이다 
그를 기반으로 핸들러를 작성했고 먼저 페이지부터 테스트를 하게 되면 문제가 발생될것이다 

415 에러가 발생되는데 이는 이는 서버에서 받게끔 유도한 데이터 타입과 , 클라이언트에서 보내는 데이터 타입의 차이가 발생해서 그렇다 
이때는 postman 을 이와 같이 수정을 해야 하는데 

![1](https://user-images.githubusercontent.com/58513678/231365586-36d71237-659c-4d4e-a0a2-48789252889e.jpg)


먼저 기존에 url - form 부분에 작성되었던 부분을 취소하고 body 의 raw 부분에 작성을 해야 한다 
그러면 postman 이 알아서 content-type 를 바꾸는데 

![2](https://user-images.githubusercontent.com/58513678/231365752-6bf3f123-e8d9-4f20-9ff0-89a7a06d7fbb.jpg)

이렇게 바꾸게 되는데 이때 content-type 은 내용의 현재 타입을 말하는것이다 그것을 application/json 형태로 변경을 하는것이고 
그것을 헤더에서 심어서 보내게 되면 서버는 이 content-type 을 읽고 이를 key-value 형태의 map 으로 읽을려고 할것이다 

이때 핸들러도 이와 같이 바뀌는데 

```

@PostMapping(value = "/id2/board" , consumes = "application/json")
public String path5 (@RequestBody Map<String , Object> requestBody){

    return requestBody.get("id").toString() + requestBody.get("boardNumber").toString();
}


```
이렇게 consumes 생기면서 이 핸들러는 content-type 이 application/json 형태인 데이터만 받겠다는 뜻이 된다 그 외로 들어오는 모든 타입은 다 거절 및 415 에러가 발생한다 
그럼 postman 까지는 ok 클라이언트에서는 어떻게 보낼까 불론 지금있는 form 으로 보내면 똑같은 문제가 발생한다 그래서 데이터를 가공을 해서 보내야 하는데 
form 문은 그럴 수 없기에 이때는 fetch , ajax 를 이용해야 한다 ajax 는 jquery 를 깔아야 함으로 pass 하고 순수 자바 스크립트로 가능한 

fetch 로 만들어볼것이다 

## fetch 요청



hello.js
```

const form_table = document.querySelector("#form-table");
const input_id = document.querySelector("#form-table .id");
const input_boardNum = document.querySelector("#form-table .boardNumber");
const button_form = document.querySelector("#form-table .form-button");



button_form.addEventListener('click' ,  (e =>{

    console.log(input_id)
    const id = input_id.value;
    const boardNum = input_boardNum.value;

    const data = {
            "id": id ,
            "boardNumber" :  boardNum
    };

    fetch('http://localhost:8080/id3/board', {
      method: 'POST', // 또는 'PUT'
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    })
    .then((response) => response.json())
    .then((data) => {
      console.log('성공:', data);
    })
    .catch((error) => {
      console.error('실패:', error);
    });



}))


```

hello.html
```

<!DOCTYPE html>
<html lang="en">
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>hello</title>
</head>
<body>

Hello MVC Project 2


<form id="form-table" action="/id/board" method="post">

    id : <input class = "id" name = "id">
    boardNumber : <input class = "boardNumber" name = "boardNumber">


    submit : <input class = "form-button" type="button" value = "제출">

</form>


<script src = "/js/hello.js"></script>
</body>
</html>

```


```

@PostMapping(value = "/id3/board" , consumes = "application/json")
public Map<String, Object> path6 (@RequestBody Map<String , Object> requestBody){

    return requestBody;
}

```
fetch 문구에대해서는 다음에 알아보고 이는 비동기 통신을 위한 순수 자바스크립트 요청이다 이전에는 ajax 를 활용했지만 jquery 플러그인을 깔아야 함으로 이제는 대세가 되어버린 
fetch 문구를 활용할것이다 

그리고 요청을 넣게 되면 결과는 

```
 .then((response) => response.json())
    .then((data) => {
      console.log('성공:', data);
    })

```
들어오게 되는데 

```
{id: 'time2', boardNumber: '21313123'}
```

이렇게 데이터 들어오는것을 확인할 수 있다