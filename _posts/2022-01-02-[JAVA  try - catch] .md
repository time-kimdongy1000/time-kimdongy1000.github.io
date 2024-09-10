---
title: JAVA  try - catch
author: kimdongy1000
date: 2022-01-03 10:00
categories: [Back-end, Java]
tags: [Generics]
math: true
mermaid: true
---


## 예외 vs 에러 
컴퓨터 공학책에서는 이 둘을 이렇게 나눕니다 

1. 예외 - 프로그래밍 단계에서 예상할 수 있는 오류 
            소스코드의 비정상적인 오류로 발생하는 사건들 이때 소스코드의 적절한 조치로 어플리케이션이 정지 되지 않고 계속해서 서비스를 이어갈 수 있음

2. 에러 - 프로그래밍 단계에서 예상할 수 없는 오류 
            프로그램문제가 아니라 외부의 요인의 문제로 발생 - 인터넷이 끊겼다 , 메모리가 부족 등등의 소프트웨어 외적인 문제로 발생하는 것 


에러가 발생하면 프로그램 외적인 이유로 발생한것이기 때문에 단순 소스코드로 오류를 발견해서 넘어가는것이 매우어렵습니다 그래서 프로그래밍에서는 
오류보다는 예외에 집중에서 프로그램을 만들게됩니다 

## try - catch

java 문법에서 예외가 발생할것을 예상해서 예외발생시 어떤 행동을 취하라는 뜻입니다 만약 예외처리 없이 소스가 실행이 되어 예외상황을 마주하게 되면 시스템은 심각한 문제를 일으키고 정지하게 됩니다 

```
package test;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;

public class TryCatchTestClass2 {

    public static void main(String[] args) {

        List<Integer> int_array = new ArrayList<>();
        int_array.add(0);
        int_array.add(1);
        int_array.add(2);
        int_array.add(3);

        Random random = new Random();

        for(int i = 0; i < 10; i++){

            int rnd_index = random.nextInt(10);
            System.out.println(int_array.get(rnd_index));
            int div = rnd_index / int_array.get(rnd_index);
        }
    }
}


```
예를 들어서 다음의 코드가 있다고 생각을 해봅시다 이 코드의 문제점은 다음과 같습니다 

1. 배열의 전체 길이보다 긴 참조값에 접근할려고 함

2. 어떤 수를 0으로 나눌려고 함

이 두개의 예외는 프로그램에서 심각하게 오류를 일으키는 심각한 코드들입니다 그렇기에 개발자는 이런 예외를 예상하여 프로그램이 정지되지 않도록 신경을 써야합니다 

## try - catch - finally

```
public class TryCatchTestClass {

    public static void main(String[] args) {

        try{

            List<Integer> int_array = new ArrayList<>();
            int_array.add(0);
            int_array.add(1);
            int_array.add(2);
            int_array.add(3);

            Random random = new Random();

            for(int i = 0; i < 10; i++){

                int rnd_index = random.nextInt(10);
                 System.out.println(int_array.get(rnd_index));
                 int div = rnd_index / int_array.get(rnd_index);
            }

        }catch(ArrayIndexOutOfBoundsException e1){
            e1.printStackTrace();

        }catch(Exception e2){
            e2.printStackTrace();
        }finally {
            System.out.println("finally");
        }

    }
}


```
java 에서 대표적으로 예외를 관리하고 처리할 수 있는 문구는 바로 try - catch - finally 입니다 

try 는 예외가 예상이 가는 곳에 소스코드를 작성을 해두고

catch 부분에는 해당 예외가 발생했을때 어떤 조치를 취할것인지에 대한 소스코드를 적어둡니다 (보통 catch 에서는 오류를 감지해서 기록하는 log 에 관련한 소스가 많습니다)

finally 는 try - catch 문구에 상관없이 항상 실행되는 최종문구입니다 (try 에서 발생한 자원을 회수하는 기능을 주로 사용합니다)

## try-with-resources
예전 1.6 JDK 버전에서는 항상 finally 에 자원을 회수를 해주어야 한다 DB 커넥션 네트워크 파일 관련한 객체들에 말이다 하지만 JDK 1.7 부터는 try-with-resources 라는 문구를 사용해서 굳이 finally 에 자원회수 없이 바로 문구가 끝이나면 자동으로 자원을 회수하게 만들어졌다 

```
public class TryWithResourcesExample {
    public static void main(String[] args) {
        try (BufferedReader br = new BufferedReader(new FileReader("test.txt"))) {
            String line;
            while ((line = br.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            System.out.println("파일을 읽는 중에 오류가 발생했습니다: " + e.getMessage());
        }
    }
}

```
원래 같으면 catch 다음에 finally 에 BufferedReader 대한 자원을 회수해야 하는 로직을 마저 작성을 해주어야 했다 

## Exception 
예외라는 것으로 이 예외는 크게 2가지로 나뉩니다 하나는 Checked Exception (검사된 예외) ,  Unchecked Exception (검사되지 않은 예외) 이 둘의 가장 큰 차이점은 

Checked Exception 은 컴파일 시점에 예외르 어떻게 처리할것인지에 대해서 먼저 정의를 해주어야 합니다 그렇지 않으면 IDE 가 붉은색으로 이를 경고합니다 

Unchecked Exception 는 런타임 시점에 발생하는 예외로 컴파일시점에서는 발생하지 않을 수 있지만 실행시점에서는 발생할 수 있는 예외입니다 

지금 보면 
```
for(int i = 0; i < 10; i++){

    int rnd_index = random.nextInt(10);
    System.out.println(int_array.get(rnd_index));
    int div = rnd_index / int_array.get(rnd_index);
}
```
이 부분에서 2가지 예외가 발생할 수 있는데 모두 검사되지 않은 예외 즉 컴파일 단계에서는 발생하지 않을 수도 있는 예외입니다 다만 우리가 볼때는 명확해 보입니다 배열의 길이보다 더 큰 값을 참조할 수 있고 어떤 수를 0으로 나눌수 있습니다 다만 IDE 는 이를 예외를 강제하지 않는다고 판단해서 굳이 예외를 선언적으로 강제하지는 않습니다 

다만 아래와 같은 경우는 컴파일 단계에서 예외를 처리를 강제하게 됩니다 

```
public class CheckedExceptionExample {
    public static void main(String[] args) {
        try {
            FileReader file = new FileReader("test.txt"); // FileNotFoundException이 발생할 수 있음
        } catch (FileNotFoundException e) {
            System.out.println("파일을 찾을 수 없습니다: " + e.getMessage());
        }
    }
}
```
이처럼 어떤 파일을 처리할려고 할때 IDE 는 이때 명확하게 파일이 없을 수 있는 예외를 예상하고 개발자에게 IDE 경고를 예외문구를 강제하게 됩니다 

## 예외의 종류
지금여러 소스를 보았을때 우리는 여러가지의 예외를 보고 있습니다 

1. ArrayIndexOutOfBoundsException - 배열길이를 초과해서 참조할려고 할때 발생하는 예외

2. FileNotFoundException - 파일이 없는데 파일을 참조할려고 할때 발생하는 예외

3. ArithmeticException - 어떤 수를 0으로 나눌려고 할때 발생하는 예외

4. Exception - 모든 예외를 아우르는 최상위 객체 

## Spring - mvc 
그럼 같은 예제를 Spring MVC 에서 해보도록 하자

```
@RestController
public class TryCatchController {

    @GetMapping("/try-catch")
    public ResponseEntity<?> testApi()
    {


        List<Integer> int_array = new ArrayList<>();
        int_array.add(0);
        int_array.add(1);
        int_array.add(2);
        int_array.add(3);

        Random random = new Random();

        for(int i = 0; i < 10; i++){

            int rnd_index = random.nextInt(10);
            System.out.println(int_array.get(rnd_index));
            int div = rnd_index / int_array.get(rnd_index);
        }

        return new ResponseEntity<>(null , HttpStatus.OK);

    }
}


```

이런 소스가 있다고 하자 현재 예외가 발생할것이라고 예상되는 2가지 포인트가 있다 하나는 ArrayIndexOutOfBoundsException , ArithmeticException 이다 프로그램 특정산 예외가 발생했을때 이를 처리하지 않으면 프로그램은 비정상적으로 종료가 된다 하지만 단순 main 에서는 예외 발생시 프로그램이 종료가 되었지만 mvc에서는 그렇지 않다 그 이유는 
MVC 프레임워크의 고유한 예외처리 메커니즘이 있기 때문이다 자세한 설명은 다음에 하고 핵심은 

Controller 레벨에서 예외가 발생하더라도 스프링 프레임워크의 고유한 글로벌 예외처리 시스템으로 예외상황이 맞이하더라도 예외를 적절하게 응답 메세지로 만들어서 보낸다음 
프로그램은 다음과 계속해서 사용할 수 있게 만들어두었습니다 

## 무조건 RuntimeException 으로 다시 던져라

```
@RestController
public class TryCatchController {

    private static final Logger log = LoggerFactory.getLogger(TryCatchController.class);

    @GetMapping("/try-catch")
    public ResponseEntity<?> testApi()
    {

        try {


            List<Integer> int_array = new ArrayList<>();
            int_array.add(0);
            int_array.add(1);
            int_array.add(2);
            int_array.add(3);

            Random random = new Random();

            for (int i = 0; i < 10; i++) {

                int rnd_index = random.nextInt(10);
                System.out.println(int_array.get(rnd_index));
                int div = rnd_index / int_array.get(rnd_index);
            }
        }catch (ArrayIndexOutOfBoundsException e1) {
            log.error(e1.toString());
            throw new RuntimeException(e1);
        }catch(ArithmeticException e2){
            log.error(e2.toString());
            throw new RuntimeException(e2);
        }catch(Exception e){
            log.error(e.toString());
            throw new RuntimeException(e);
        }

        return new ResponseEntity<>(null , HttpStatus.OK);

    }
}


```
지금 코드는 try - catch 안에 예상되는 예외를 적고 예외가 발생시 RuntimeException 으로 다시 던지는 예외입니다 나는 개인프로젝트를 할때 이런 식으로 예외를 처리했습니다 
하지만 이러한 처리방식은 원래 발생한 예외를 없애고 다시 RuntimeException 이라는 예외를 만들어서 보내게 되는 것입니다 즉 어떤 예외가 발생해도 최종적으로 클라이언트가 받아보는 예외는 RuntimeException 입니다 이러한 예외처리는 개발하는 입장에서는 쉽지만 운영 단계에서는 예외의 종류가 무엇때문에 발생하는지 알수가 없습니다 그래서 추천하는 방식은 다음과 같습니다

## 확실한 예외처리

```
@ControllerAdvice
public class ControllerAdviceHandler {


    @ExceptionHandler(RuntimeException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity<?> runtimeHandleException(Exception e) {

        Map<String , Object> resultMsg = new HashMap<>();
        resultMsg.put("Throwable" , "RuntimeException");
        resultMsg.put("errorMsg" , e.getMessage());

        return new ResponseEntity<>(resultMsg ,  HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(ArrayIndexOutOfBoundsException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity<?> arrayIndexOutOfBoundsException(Exception e) {

        Map<String , Object> resultMsg = new HashMap<>();
        resultMsg.put("Throwable" , "arrayIndexOutOfBoundsException");
        resultMsg.put("errorMsg" , e.getMessage());

        return new ResponseEntity<>(resultMsg ,  HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(ArithmeticException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity<?> arithmeticException(Exception e) {

        Map<String , Object> resultMsg = new HashMap<>();
        resultMsg.put("Throwable" , "arithmeticException");
        resultMsg.put("errorMsg" , e.getMessage());

        return new ResponseEntity<>(resultMsg ,  HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

```

```
@RestController
public class TryCatchController {

    private static final Logger log = LoggerFactory.getLogger(TryCatchController.class);

    @GetMapping("/try-catch")
    public ResponseEntity<?> testApi()
    {

        try {


            List<Integer> int_array = new ArrayList<>();
            int_array.add(0);
            int_array.add(1);
            int_array.add(2);
            int_array.add(3);

            Random random = new Random();

            for (int i = 0; i < 10; i++) {

                int rnd_index = random.nextInt(10);
                System.out.println(int_array.get(rnd_index));
                int div = rnd_index / int_array.get(rnd_index);
            }
        } catch(Exception e){
            log.error(e.toString());
            throw e;
        }

        return new ResponseEntity<>(null , HttpStatus.OK);
    }
}

```
확실한 예외처리는 다음과 같습니다 여러개의 try - catch 를 사용하지 말고 이곳에서는 하나의 부모객체 Exception 을 사용하고 이때 에러를 다시 RuntimeException 으로 감싸지 말고 
에러자체를 `throw e;` 로 던지면

`@ControllerAdvice` 에서 예외처리를 다음과 같이 특정 정해진 핸들러로 응답메세지를 받을 수 있게 작성해서 보낼 수 있습니다 

```
{
    "Throwable": "RuntimeException",
    "errorMsg": "Index 6 out of bounds for length 4"
}

{
    "Throwable": "arithmeticException",
    "errorMsg": "/ by zero"
}

```
이런식으로 말입니다 그래서 다시금 에러를 RuntimeException 으로 감싸는것보다는 그냥 에러 자체를 던져서 프레임워크 또는 지금과 같은 controllerAdvise 를 쓰는것을 추천드립니다 
