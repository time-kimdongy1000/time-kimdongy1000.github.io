---
title: JAVA  람다함수와 지역변수
author: kimdongy1000
date: 2022-01-01 12:00
categories: [Back-end, Java]
tags: [Files_walkFileTree]
math: true
mermaid: true
---

## JAVA 화살표 함수를 사용할때 마딱드리는 상황 

1. 예를들어서 다음과 같은 상황이 있다 

```
public class LambdaPractices {



    public static void main(String[] args) {

        List<Integer> originArray = Arrays.asList(1 , 2, 3, 4, 5, 6, 7, 8, 9 , 10);
        originArray.forEach(System.out::print);

        int sum = 0;
        List<Integer> multiple_2_array = originArray.stream().map(x -> {

            sum+=x;

            return x * 2;
        }).collect(Collectors.toList());
    }
}


```
이 상황은 현재 1부터 10까지의 배열을 꺼내서 곱하기 2를 해준 새로운 배열을 만드는데 이때 sum 을 해서 기존배열의 원래 합게를 갖고 있을려고 하는 예제이다 문제는 이곳에서 
람다 함수 안에 있는 sum 은 사용할 수 없다고 알려준다 

`Variable used in lambda expression should be final or effectively final`
람다 표현식 안에 있는 변수는 final 이거나 final 처럼쓰여야 한다는 것이다 여기서 final 은 아시다 싶히 한번 고정되면 변경할 수 없는 변수이다 

그래서 바깥의 변수 선언에서 

```

final int sum = 0;
List<Integer> multiple_2_array = originArray.stream().map(x -> {

    sum+=x;

    return x * 2;
}).collect(Collectors.toList());


```

이렇게 바꿔준다고 한들 역시나 에러가 발생한다 

`Cannot assign a value to final variable 'sum'` final 값에 새로운 값을 넣을 수 없다는 것이다 즉 우리가 눈으로 코드를 볼때에는 당연히 동작하는 코드여야 하지만 
어떤 방식으로 보든 이런 방식으로 원래 배열의 합계를 구할 수 없다 왜 그런지 알아보자 

이는 기본적으로 람다라는 것은 기본적으로 함수형 프로그래밍의 개념을 따라야 한다는 것입니다 이 개념은 각각의 연산이 입력값에 대한 변환을 수행하고 외부상태를 변경하지 않아야 합니다 
그런 의미에서 보면 바깥에 선언된 sum 변수를 람다 안에서 변경을 하고 있어 이는 순수함수의 상태를 깨트린다는 것입니다 이건 프로그램 패러다임에 대한 이야기고 아키텍쳐 관점으로 보게 된다면

## Local Capturing lambda
람다표현식에서 외부의 지역변수를 캡쳐한다는 뜻입니다 람다표현식은 내부지역변수를 캡쳐한다는 뜻입니다 그 뜻은 아래 소스를 보면 

```
final int value = 10;
Runnable runnable = () -> {
    System.out.println("Captured value: " + value);
};

runnable.run();

```
지금 보면 바깥 변수 value 에 대해서 람다 함수가 가져다 쓰고 있는 모습을 볼 수 있습니다 이를 지역변수의 캡쳐 다른 뜻으로 표현하면 람다 내부에 있는 value 라는 변수는 외부 변수 value 의 복사본입니다 즉 복사된 지역변수는 람다내부에서 변환할 수 없기 때문에 이때는 final 이라는 것을 사용해서 변수값이 람다 내부에서 수정된느 것을 막는것입니다 

## 실행되는 메모리 위치 
지역변수는 스택이라는 메모리 공간에 할당이 됩니다 반대로 람다표현식 그리고 내부에서 사용하는 지역변수 (캡쳐변수 포함) 는 힙 영역에 생성이 됩니다 

1. 람다표현식은 메서드의 레퍼런스나 인터페이스 구현을 통해 객체로 저장이 됩니다 

2. 이때 람다표현식이 외부 변수를 캡쳐할때 이 외부 변수는 힙메모리에 저장이 됩니다 

3. 이렇게 객체로 만들어진 이후에는 더이상 해당 스코프에 종속이 되지 않으며 람다 객체가 유효한 동안 캡쳐한 값은 유지가 됩니다 

## 서로 다른 쓰레드 
지역변수를 관리하는 쓰레드와 , 람다를 관리하는 쓰레드가 다를 수 있다 이때 전달받은 캡쳐 변수만 사용이 가능할뿐 값을 재할당하는것은 쓰레드가 다른 쓰레드에 있는 지역변수를 접근하는 것을 
막고 있기 때문이 이것이 아니라면 모든 쓰레드를 싱크를 관리를 해주어야 하는데 사실상 불가능한 이야기다 

## 예외 java - script

```
const originArray = [1 , 2, 3, 4, 5, 6, 7, 8, 9 , 10]; 

let sum = 0;
const multiple_2_array = originArray.map( x => {
	sum += x;
	return x * 2;
});

console.log(sum)
```

똑같은 소스를 자바 스크립트로 만들었다 이때는 java 에서 동작하지 않는게 자바 스크립트 안에서는 동작이 된다 왜 이런것일까? 이때는 자바스크립트의 특수 경우이고 이떄는 스코프 체인과  클로저의 영향이다 다음을 따라가보자

1. map 안에서 외부 변수 sum 을 참조하고 값을 변경할려고 시도 

2. sum 이라는 변수는 현재 내부에 존재하지 않으므로 스코프 체인을 따라서 올라감 

3. map 이 선언된 바로 위에 sum 이라는 변수를 찾았음 

4. 원래 map 은 순수함수가 보장되는 함수지만 현재 외부 변수를 수정함으로서 이는 순수함수가 아니게 됨

오늘은 왜 java 람다식에서 외부 변수를 호출은 가능하지만 값변경이 안되는지에 대해서 알아보았습니다 그와 반대로 자바 스크립트는 왜 되는가에 대해서 알아보았습니다 
저도 개발을 하다보면 자바 똑같은 함수형 프로그램인데 어떤곳은 되고 안되고 하니 한번 정리가 필요할 꺼 같아서 정리를 해보았습니다