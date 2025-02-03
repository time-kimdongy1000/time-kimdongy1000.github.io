---
title: Spring MVC WebSocket
author: kimdongy1000
date: 2023-04-20 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC , WebSocket ]
math: true
mermaid: true
---

## WebSocket 란
WebSocket 은 서버와 클라이언트 간 양방향 통신을 지원하는 프로토콜입니다 일반적은 HTTP 프로토콜하고는 다르게 클라이언트와 서버 간의 연결을 유지하면서 데이터를 주고받을 수 있습니다

HTTP는 Request - Response 가 끝이 나면 연결이 끊어지게 됩니다 우리는 이 글을 통해서 WebSocket 기본 연결을 포함해서 점점 복잡한 프로그램을 만들어갈 것이다 다만 HTTP 연결만 해보았던 사람이 바로 웹소캣을 알기에는 많이 힘들기에 천천히 간단한 예제를 보면서 하나씩 만들어보겠습니다

## 사전작업 
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


<!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.24</version>
    <scope>provided</scope>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-websocket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>


```
다음과 같이 의존성을 주입해주고


## webSocket html 
```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>웹소캣 테스트</title>
</head>
<body>

    <div>
        <label for="name">Name:</label>
        <input type = "text" id = "name"/>
        <br/><br/>

        <label for="message">Message:</label>
        <input type = "text" id = "message"/>
        <br /><br/>

        <button onclick="send()">send</button>
    </div>
    <hr/>
    <div id = "messages"></div>




<script src = "/js/stomp.js"></script>
<script src = "/js/sockjs.min.js"></script>
<script>
    let stompClient = null;
    const messages_div = document.querySelector("#messages");
    const name_input = document.querySelector("#name");
    const message_input = document.querySelector("#message");



    function connect () {

        // ws 엔드포인트로 연결요청
        let socket = new SockJS('/ws');

        //over 메서드를 사용해서 Stomp Broker 에 연결
        stompClient = Stomp.over(socket);


        // connect 연결을 통해서 연결 이때 연결하는 구독 정보는 /topic/messages 로 연결
        // 그러면 이 정보는 @SendTo("/topic/messages") 핸들러와 연결 하게 됩니다
        // 그리고 메세지가 도착하면 자동으로 호출되며 이때 essage.body 의 정보가 들어가게 됩니다
        stompClient.connect({} , (frame) => {
            console.log('Connected:' + frame)
            stompClient.subscribe('/topic/messages' , (message) => {
                showMessage(JSON.parse(message.body));
            })
        })
    }

    // 들어오는 메세지가 있으면 메세지 창을 만드는거 부터 시작
    function showMessage( message ){

        console.log(message);

        let messageElem = document.createElement("div");
        let messageSender = document.createElement("strong");
        messageSender.innerText = message.sender + ":" + message.content;

        messageElem.appendChild(messageSender);

        messages_div.appendChild(messageElem);

    }

    // 메세지 보내는 곳 이때 send 라는 메서드를 사용하며 이때 /app 접두어가 붙는 이유는
    // registry.setApplicationDestinationPrefixes("/app"); 를 사용하기 때문입니다
    function send(){

        let name_value =  name_input.value;
        let message_value = message_input.value;

        stompClient.send("/app/chat" , {} , JSON.stringify({sender : name_value , content : message_value}))
        message_input.value = "";

    }

    connect();




</script>
</body>
</html>
```

WebSocket html 은 간단하게 만들었다 이때 사용한 라이브러리는 stomp , sockjs를 사용했다
간단하게 이 라이브러리를 설명하면

STOMP (Stopm Over WebSocket)  WebSocket 위에서 동작하는 메시지 프로토콜입니다 WebSocket는 어플리케이션 양방향 통신을 가능하게 하지만 메시지의 구조와 전송 방법에 대한 정의가 없습니다
그래서 STOMP 라이브러리는 WebSocket을 기반으로 하며 클라이언트와 서버 간의 표준화된 메시지 교환을 가능하게 합니다

STOMP  메시지는 헤더, 메시지 본문 및 옵션 프레임으로 구성

그리고 SockJS는 WebSocket 이 지원하지 않는 브라우저에서도 WebSocket 이 동작할 수 있는 javaScript 라이브러리입니다 주석은 다 달려 있으니 참고하면 되고 이제 백엔드 프로그램을 만들어보자



## WebSocketConfig.java

```

package com.cybb.main.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {

        /**
         * enableSimpleBroker 메서드는 보통 두가지 타입이 존재하는데
         *
         *  /topic
         *  한명이 message 를 발행했을 경우 구독하고 있는 n명 모두에게 메세지를 보냄
         *  (공개 채팅방)
         *
         * /queue
         * 한명이 message 를 발행했을때 발행한 한명에게 메세지가 되돌아와야 하는 경우 일대일 구독으로
         * 가장 먼저 메세지를 받는 사람만 유일한 클라이언트가 되고 다른 클라이언트는 받을 수 없습니다
         * (이메일 같은거 )
         *
         *
         *
         * */

        registry.enableSimpleBroker("/topic");


        registry.setApplicationDestinationPrefixes("/app");
    }

    /**
     * 웹소캣 연결시 사용할 엔드포인트
     *
     * html let socket = new SockJS('/ws'); 와 연결됨
     *
     * */
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}


```

## WebSocketController.java

```

package com.cybb.main.controller;

import com.cybb.main.dao.ChatMessage;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class WebsocketController {

    @GetMapping("/webSocket")
    public String goWebSocketPage(){

        return "webSocket";
    }


    @MessageMapping("/chat")
    @SendTo("/topic/messages")
    public ChatMessage send(ChatMessage message) throws Exception
    {
        Thread.sleep(1000);

        System.out.println("sender" + message.getSender());
        System.out.println("message" + message.getContent());
        return new ChatMessage(message.getSender() , message.getContent());

    }
}


```

## ChatMessage.java
```
package com.cybb.main.dao;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Builder
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ChatMessage {

    private String sender;

    private String content;
}


```

config는 자세히 읽어보면 주석을 다 잡아넣었으므로 참고를 하면 되고 다음은 WebSocketController 우리가 중요하게 봐야 할 소스 코드이다  이 소스 코드에서 두 개의 핸들러가 존재하는데

```

@GetMapping("/webSocket")
public String goWebSocketPage(){

    return "webSocket";
}

```

첫 번째 핸들러 이것은 그냥 페이지 이동이다

그리고 두 번째 핸들러가 중요한데

```

@MessageMapping("/chat")
@SendTo("/topic/messages")
public ChatMessage send(ChatMessage message) throws Exception
{
    Thread.sleep(1000);

    System.out.println("sender" + message.getSender());
    System.out.println("message" + message.getContent());
    return new ChatMessage(message.getSender() , message.getContent());

}


```

우리가 html에서 메시지를 보내면
`stompClient.send("/app/chat" , {} , JSON.stringify({sender : name_value , content : message_value}))` 이 요청으로 들어가게 된다 이 chat 핸들러는 위의

`@MessageMapping("/chat")`으로 들어오게 되는데 이때 앞의 chat 이 없어서 안될 수 있다고 생각할 수 있지만

우리가 앞에서 config 파일 작성할 때 `registry.setApplicationDestinationPrefixes("/app");` 이렇게 해서 작성한 것을 본 적이 있을 것이다 이렇게 하면 앞의 메시지를 보낼 때 핸들러 chat를 생략할 수 있다

`@SendTo("/topic/messages")` 이 어노테이션은 결국 받은 메시지를 누구에게 보내냐 인 것인데 이때는 이 메시지를 구독하고 있는즉 우리가 처음

```
stompClient.subscribe('/topic/messages' , (message) => {showMessage(JSON.parse(message.body));})

``` 
를 본 적이 있을 것이다 이 요청을 기반으로 채팅창을 나누게 되는 것이고 메시지를 독립적으로 받게 되는 것이다 그럼 간단한 시연과 함께 포스트를 종료하겠습니다

