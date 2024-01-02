---
title: Spring MVC 와 HTTP
author: kimdongy1000
date: 2023-04-12 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC  HTTP ]
math: true
mermaid: true
---

## Spring - MVC 
Spirng MVC 는 웹 애플리케이션을 개발하기 위한 인기있는 프레임워크중 하나입니다 MVC 를 활용하면 모델 - 뷰 - 컨트롤러 아키텍쳐를 기반으로 개발을 진행을 하게 됩니다 

1. 모델 (Model)
    웹 어플리케이션에서 데이터와 비즈니스 로직을 담당하는 계층입니다 모델은 컨트롤러와 , 뷰 사이에서 데이터를 전달하고 관리하는 역활을 수행하게됩니다 

2. 뷰 (View)
    웹 어플리케이션에서 최종적으로 사용자에게 화면으로 보여지는 부분입니다 spring 에서는 주로 html , jsp , thyleamf 가 주로 사용됩니다 

3. 컨트롤러 (Controller)
    웹 어플리케이션에서 사용자의 요청을 처리하고 그에 따른 응답을 생성하는 역활을 하게 됩니다 컨트롤러는 모델과 뷰 사이의 중간 매개체로서 모델에서 처리한 데이터 결과를 뷰로 전달하는 역활을 하게 됩니다 

이 3가지를 합쳐서 모델 뷰 컨틀롤러 아키텍쳐라고 하게 됩니다 

## DispatcherServlet
MVC 핵심 구성중 하나로서 클라이언트의 HTTP 요청을 받아서 이를 적절한 컨트롤러로 보내고 받은 결과를 뷰에 전달하여 응답을 생성하는 역활을 하는 MVC 의 심장같은 존재입니다 

1. 클라이언트 요청의 분석 및 처리 , Hanlder Mapping 
    모든 HTTP 요청은 DispatcherServlet 에처 처리하게 됩니다 클라이언트의 url 을 분석하여 어느 컨트롤러로 전달할지 결정하는 역활을 맡습니다 이때 Handler Mapping 는 컨트롤러와 요청 url 간의 매핑을 제공합니다 

2. Handler Adapter 
    선택된 컨트롤러에는 적절한 Handler Adapter 이 적용됩니다 이는 컨트롤러가 어떤 요청을 어떻게 처리할지에 대한 규칙을 정하게 됩니다 

3. 뷰의 선택과 모델 데이터 전달 , View Resolver 
    컨트롤러가 실행된 후 DispatcherServlet 은 뷰를 선택하고 모델 데이터를 뷰에 전달합니다 이를 통해 클라이언트에게 응답을 생성합니다 
    그리고 View Resolver 를 통해 뷰 이름을 물리적인 뷰 템플릿 파일 경로로 변환합니다 이는 뷰의 논리적인 이름과 물리적인 위치를 매핑합니다 

4. 뷰의 렌터딩
    선택된 뷰는 모델 데이터와 함께 렌더링되어 최종적인 응답을 생성합니다 이 응답은 클라이언트로 전송되어 화면에 표기 됩니다 

이 밖에도 몇가지 더 있지만 DispathcerServlet 의 핵심적인 역활은 요청에 대한 응답생성 이 핵심이다 

## DispatcherServlet 구경하기 
boot 이전에서는 DispatcherServlet 은 거의 기본적인 기능만 제공이되었고 필요하면 개발자가 직접 그 기능에 대한 것들을 붙여 놓아야 했다 다만 boot 로 넘어오면서 그런 자잘한 구성까지도 이미 구현이 되어 있는채로 사용이 가능하다 

```
public class DispatcherServlet extends FrameworkServlet {

    protected void initStrategies(ApplicationContext context) {
		
        initMultipartResolver(context); /*파일 다운로드 업로드와 관련한 설정*/
		
        initLocaleResolver(context); /*다국어를 설정과 관련한 설정 */
		
        initThemeResolver(context); /*웹 테마와 관련한 설정*/
		
        initHandlerMappings(context); /*클라이언트 url 과 컨트롤러 매핑을 위한 설정*/
		 
        initHandlerAdapters(context); /*매핑된 컨트롤러가 어떤 규칙으로 동작할지에 대한 설정 */
		
        initHandlerExceptionResolvers(context); /*발생된 에어를 어떻게 처리할것인지에 대한 설정 */
		
        initRequestToViewNameTranslator(context);  /*클라리언트에게 어떤 뷰를 보여줄지 결정하는 설정 */
		
        initViewResolvers(context); /*컨트롤러를 통해서 렌더링될 논리적인 뷰와 물리적 파일 뷰의 매핑을 위한 설정*/
		
        initFlashMapManager(context); /*리다이렉션과 관련한 설정 */
	}
}

```
이 부분이 기본설정이다 아무런 설정이 없으면 이 DispatcherServlet 는 이 기본적인 설정을 따르게 되며 아마 여러분이 웹 애플리케이션을 만들때에도 이 범위를 벗어나지는 않을것입니다 

그럼 spring mvc 를 직접 만드는것은 다음시간부터 하고 이 이후는 HTTP 에 대한 이야기를 조금 할것이다 이 내용은 웹개발을 하는 사람이라면 그래도 어느정도는 알고 있어야 한다 

## HTTP 란
인터넷상에서 클라이언트와 서버가 정보를 주고 받기 위한 표준 프로토콜 중 하나입니다 웹으로 통신하는 데이터 규약이며 이 통신규약은 약속이며 어떤 웹프로그램든 이 규약을 벗어날 수 없습니다 

## HTTP 요청 (Request)
```
Request URL:
https://www.google.com/search?q=%EA%B0%9C&oq=%EA%B0%9C&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTINCAEQABiDARixAxiABDIKCAIQABixAxiABDINCAMQABiDARixAxiABDINCAQQABiDARixAxiABDINCAUQABiDARixAxiABDIGCAYQABgDMg0IBxAAGIMBGLEDGIAEMg0ICBAAGIMBGLEDGIAEMgYICRAAGAPSAQkxMzE3ajBqMTWoAgCwAgA&sourceid=chrome&ie=UTF-8

Request Method: GET

Status Code: 200 OK

Remote Address: 142.250.206.196:443

Referrer Policy: strict-origin-when-cross-origin

```
이게 정확한 모양은 아니지만 이런 모양으로 요청 또는 응답이 만들어집니다 자세한 모양은 아래에서 다루게 됩니다 


## HTTP 응답 (Response)
```
Content-Type:
text/html; charset=UTF-8
Cross-Origin-Opener-Policy:
same-origin-allow-popups; report-to="gws"
Date:
Thu, 28 Dec 2023 13:01:44 GMT

<!doctype html>
<html itemscope="" itemtype="http://schema.org/SearchResultsPage" lang="ko">
    <head>
    ...
    </head>
    ...
<body> 
    ...
</body>
```
위의 요청에 대한 응답입니다 응답은 헤더를 포함한 클라이언트에게 보여줄 body 를 참조하게 됩니다 


## HTTP 전체적인 흐름 

1. 어플리케이션에서 HTTP 요청 메세지 생성 
    GET /search?q=youtube&.... http/1.1
    HOST : www.google.com  

2. TCP 세그먼트 씌우기 
    TCP 세그먼트는 HTTP 요청이라고 해도 안에 TCP 전송계층이 존재합니다 (이를 IP 4계층이라고 하는데) 이때 1번에서 만들어진 HTTP 메세지에 부가적인 작업을 진행을 하게 되는데 
    
    ii. 데이터 전송 : 1번에서 넘어오는 데이터를 받아서 신뢰성있는 방식으로 목적지로 전달 
    ii. 흐름제어  : 순서제어 데이터를 전송하거나 응답해서 받은 데이터를 순서를 지정해서 무작위로 가더라도 받는 곳에서 재조립하게 시퀀스를 할당 
    ii. 신뢰성 : 데이터 전송시 손실을 최소화 하고 필요시 처음부터 전송을 하게 됩니다 

    등 역활을 TCP 세그먼트에서 진행을 하게 됩니다 

3. 전송 
    2번까지 완료되면 이제 데이터를 L2 , L1 (IP 4계층) 을 통해 전송 이떄 클라이언트와 구글은 다이렉트로 연결이 되어 있는게 아니라서 수많은 다른 서버를 거쳐가게 됩니다 
    그래서 목적지에 도착

4. 해석 
    요청을 받은 HOST 는 들어온 HTTP 메세지를 분석을 하게 됩니다 

5. 서버에서 HTTP 응답 메세지 생성 
    4번에서 해석이 끝이 났으면 이를 통해서 적절한 응답을 만들게 됩니다 
        
        
        Content-Type:
        text/html; charset=UTF-8
        
        <!doctype html>
        <html itemscope="" itemtype="http://schema.org/SearchResultsPage" lang="ko">
        <head>
        ...
        </head>

6. 5번 작업을 다시 L1 , L2 를 통해서 클라이언트로 전송 (이때 TCP 세그먼트에는 출발지 와 목적지가 적혀 있으므로 이를 통해서 요청자로 응답을 보낼 수 있습니다)

7. 6번에도 도착한 응답을 적절하게 렌더링

이런 무수한 과정을 통해서 우리가 보낸 요청이 응답으로 돌아오게 되는 것입니다 

## HTTP 요청 메세지 
요청메세지는 다음과 같은 패턴으로 만들어지게 됩니다 

```
[시작라인]
[헤더라인]
[공백라인]
[본문라인]

```

```

POST /example HTTP/1.1
Host: www.example.com
Content-Type: application/json
Content-Length: 26

{"username": "john_doe", "age": 28}

```
요청메세지는 이렇게 만들어집니다 첫번쨰줄이 시작라인 두밴쨰 줄은 헤더라인이고 헤더라인이 끝나서 비어있는 지점 (Content-Length: 26) 다음 라인이 공백라인 그 아래 라인이 본문라인입니다 공백라인의 비어있는 줄로 인해서 헤더라인과 본문라인을 구분할 수 있게 해놓았습니다 

## HTTP 응답 메세지
```
[시작라인]
[헤더라인]
[공백라인]
[본문라인]

```

```

HTTP/1.1 200 OK
Date: Wed, 30 Nov 2023 12:00:00 GMT
Server: Apache/2.4.41 (Unix)
Content-Type: text/html; charset=UTF-8
Content-Length: 137

<!DOCTYPE html>
<html>
<head>
    <title>Example Page</title>
</head>
<body>
    <h1>Hello, World!</h1>
</body>
</html>

```
응답 메세지도 규격은 동일합니다 첫줄이 시작라인이고 헤더라인은 시작라인 다음줄 부터해서 공백라인 전까지를 헤더라인이라고 하고 한줄을 띄운 공백라인 다음에 본문라인이 오고 있는 모습을 볼 수 있습니다 

오늘은 여기 까지 Spring MVC 시작과 그 시작전에 HTTP 통신과 관련해서 조금 정리를 해보았습니다