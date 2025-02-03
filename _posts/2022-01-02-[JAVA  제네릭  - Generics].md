---
title: JAVA  제네릭 (Generics)
author: kimdongy1000
date: 2022-01-02 10:00
categories: [Back-end, Java]
tags: [Generics]
math: true
mermaid: true
---

내가 예전에 한번 쓰다가 혼난 코드를 한번 보여주겠다 

```
@GetMapping("")
public ResponseEntity<?> resultData(){

	return new ResponseEntity<>(null , HttpStatus.OK);
}
```

팀장님이 혼낸 코드는 바로 제네릭 안에 물음표였다 팀장님이 나를 혼낸 이유는 java 문법 특성상 최상위 제네릭 타입인 물음표를 사용해도 되지만 후에 이를 인수인계할 때는 정확하게 return 할 타입을 명시해 주는 것이 맞는다고 말이다 나는 당연히 수긍했다 사실 귀찮기도 했다 매번 return 할 타입이 달라질 때마다 최상위 제네릭 타입으로 return 을 하면 아무 문제가 없기 때문에
지금은 <> 쓸 때는 타입을 명확하게 정의를 하고 있다

## 제네릭의 정의 
자바에서 제네릭(Generics)은 클래스 내부에서 사용할 데이터 타입을 외부에서 지정하는 기법을 의미한다

말부터 여러워 보이는데 바로 코드를 보겠습니다 

## Box.java
```
public class Box<T> {

    private T item;

    public void setItem(T item) {
        this.item = item;
    }

    public T getItem(){
        return item;
    }

    @Override
    public String toString() {
        return "Box{" +
                "item=" + item +
                '}';
    }
}
```

지금 보면 난생처음 보는 무엇인가 있을 것입니다 바로 T입니다 이는 타입으로 제네릭 정의를 다시 읽어보면 클래스 내부에서 사용할 데이터 타입을 외부에서 지정
즉 현재 Box에서 사용할 클래스 타입은 정해지지 않았습니다 우리는 이를 외부에서 지정을 해줄 것입니다

## Book.java
```
public class Book {

    private String name;

    private int number;

    public Book(String name, int number) {
        this.name = name;
        this.number = number;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getNumber() {
        return number;
    }

    public void setNumber(int number) {
        this.number = number;
    }

    @Override
    public String toString() {
        return "Book{" +
                "name='" + name + '\'' +
                ", number=" + number +
                '}';
    }
}


```
이렇게 하하고 다음과 같이 코드를 입력하게 되면 `Box<Book> book_in_box = new Box<>();` 하게 되면 Box 내부 클래스는 다음과 같이 변하게 됩니다 (물론 우리 눈에는 안 보입니다)

```
public class Box<Book> {

    private Book item;

    public void setItem(Book item) {
        this.item = item;
    }

    public Book getItem(){
        return item;
    }

    @Override
    public String toString() {
        return "Box{" +
                "item=" + item +
                '}';
    }
}


```
모두 Book에 관련한 데이터를 받을 수 있게 됩니다 그래서

```
Box<Book> book_in_box = new Box<>();
Book book1 = new Book("java" , 1);
book_in_box.setItem(book1);
System.out.println(book_in_box.getItem());

```
이렇게 설정을 하게 되면 book_in_box에는 Book 타입만 들어오게 됩니다 예를 들어서 다른 타입

```

Box<Book> book_in_box = new Box<>();

Book book1 = new Book("java" , 1);
Computer computer1 = new Computer("25332231" , 2)


book_in_box.setItem(book1);
System.out.println(book_in_box.getItem());

book_in_box.setItem(computer1); // 오류

```
즉 다른 타입은 들어올 수 없습니다 이를 통해서 제네릭의 장점은 타입 안정성 이 있습니다
즉 이미 선언한 타입 말고는 다른 타입은 들어올 수 없다는 뜻입니다

그래서 타입의 안정성을 보장해야 하는 것들 예를 들어서 List , Map 등등은 모두 이와 같이 제네릭 타입으로 내부의 타입을 외부에서 선언해서 사용합니다

## 와일드 카드 ? 
내가 혼난 코드?는 제네릭에서 와일드카드라고 합니다 이는 제네릭 타입을 좀 더 유연하게 하는데 특징은 외부에서 내부에서 사용할 타입을 설정은 하는데 어떤 특정한 타입으로 결정을 짓지는 않겠다는 뜻입니다

그럼 제가 혼났던 코드로 와봅시다 
```

@GetMapping("")
public ResponseEntity<?> resultData(){

	return new ResponseEntity<>(null , HttpStatus.OK);
}

```
이렇게 되면 이 핸들러는 body 가 어떤 타입과 상관없이 ResponseEntity class로 만 return 을 해주기만 하면 됩니다 이렇게 처음에 만든 이유는 내가 개발할 때 어떤 타입의 객체가 return 될지 모르고 먼저 핸들러를 먼저 작성을 할 때가 많아서입니다 일단?로 세팅을 해두면 어떤 타입으로 나가더라도 오류는 발생하지 않기 때문이죠 하지만 단점은 역시 사람이 소스를 보다 보면 핸들러를 통해서 무엇을 하려는지 명확해야 하는 경우가 있습니다 지금 같은 경우에는 그렇지 않은 것이죠 이것이 와일드카드 제네릭의 유연함입니다

## <? extends T> 상한 바운드 
제네릭을 하다 보면 필수적으로 나오는 2가지 상한 바운드 하한 바운드 이 두 가지를 설명을 하겠습니다 이를 제일 처음 본 곳은 시큐리티입니다

```

public User(String username, String password, Collection<? extends GrantedAuthority> authorities) {
		this(username, password, true, true, true, true, authorities);
	}

```
UserDetails를 만들 때 리스트 타입의 `Collection<? extends GrantedAuthority> authorities` 을 원하는데 이때 상한 바운드가 나오게 됩니다 이 상한 바운드의 특징은
바로 읽기 전용입니다 특정 타입 T와 그 하위 타입만 허용하는 와일드카드입니다. 즉 여기서는 GrantedAuthority 타입과 그 하위 타입만 허락을 하게 되는 것이죠 그런데 왜 읽기 전용인가?

이는 어떤 타입이 실제로 리스트에 들어가는지 알 수 없기 때문입니다 이게 무슨 말이냐?

## 상한 바운드	
```

List<? extends Number> list = new ArrayList<Integer>();
list.add(10); // 컴파일 오류
Number num = list.get(0); // 정상 작동

```
우리는 당연히 list에 값을 넣을 수 있다고 생각하지만 이렇게 하면 Number라도 오류가 발생하게 됩니다 컴파일러는 Number의 하위 타입이 들어오면 된다고 생각하지만
실제로 어떤 타입이 들어오는 건지 알 수 없기 때문에 아예 add라는 함수 자체를 막아버리는 것입니다 즉 list 가 add 되는 것이 Doulbe 타입인지 Number의 하위 타입인지 알 수 없기 때문에 컴파일러가 이를 허용하지 않게 됩니다 그래서 상한 바운드는 읽기 전용이 됩니다 그런데 읽기가 되는 이유는 당연합니다 list.get(0) 을 하게 되면 이는 당연히 Number의 하위 타입이 이미 들어온 것이라는 것을 명확히 알 수 있기 때문입니다 그렇기에 상한 바운드는 읽기만 가능합니다


## 하한 바운드 
반대로 하한 바운드는 쓰기만 가능하고 읽기가 불가능합니다 
```

List<? super Integer> list = new ArrayList<Number>();

list.add(10); // 정상 작동
Integer num = list.get(0); // 컴파일 오류
Object obj = list.get(0); // 정상 작동

```
이렇게 하게 되면 현재 list는 Integer를 포함한 상위 타입을 쓰기 할 때 동작할 수 있습니다 그렇기에 값을 추가할 때는 Integer 상위타입 `list.add(10);` 가 정상작동이 되지만
값을 읽어올 때는 그 값이 정확하게 어떤 타입인지 알 수 없습니다 그렇기에 특정 타입으로 값을 받고 있는 `Integer num = list.get(0);`는 오류가 발생하지만
모든 타입의 최상위 타입은 Object는 이런 문제에서 자유롭습니다

## 파라미터로 들어간 상한 , 하한 바운드

```
public User(String username, String password, Collection<? extends GrantedAuthority> authorities) {
		this(username, password, true, true, true, true, authorities);
	}


```
다시 돌아와서 상한 바운드, 하한 바운드 알겠는데 지금 여기 보아는 함수는 파라미터에 상한 바운드가 들어가 있습니다 이는 `GrantedAuthority` 은 인터페이스입니다 인터페이스를 구현한  예를 들어, SimpleGrantedAuthority는 GrantedAuthority의 한 구현체이며, 다양한 권한 모델을 지원하기 위해 다른 구현체들도 만들어질 수 있습니다.

즉 파라미터로 설정한 이유는 List 형태의 GrantedAuthority 이 필요한데 이때 GrantedAuthority 타입의 하위 class 들만 들어올 수 있게 안전한 타입을 만들어준 것입니다

읽기 속성으로 보게 되면 상한 바운드 특성상 새로운 객체를 입력할 수 없습니다 이미 입력된 권한 컬렉션을 User로 전달하는 것이기 때문에 생성시점에는 전달된 권한 목록이
변하지 않았다는 것을 보증합니다