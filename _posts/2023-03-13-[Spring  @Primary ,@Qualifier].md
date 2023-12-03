---
title: Spring Ioc @Primary ,@Qualifier
author: kimdongy1000
date: 2023-03-13 12:00
categories: [Back-end, Spring - Core]
tags: [IoC , '@Primary ,@Qualifier']
math: true
mermaid: true
---

## @Primary 
spring 은 기본적으로 동일한 타입의 bean 을 생성하지 않습니다 예를 들어서 다음과 같은 소스가 있다고 생각을 해봅시다 

```
public interface Printer {

    void print();
}

==============================

@Component
public class InkjetPrinter  implements Printer {

    @Override
    public void print() {

        System.out.println("InkjetPrinter");

    }
}

==============================


@Component
public class LaserPrinter  implements Printer {

    @Override
    public void print() {
        System.out.println("LaserPrinter");
    }
}


```

예를 들어서 상단에 Printer 인터페이스가 존재하고 그 아래 각각 InkjetPrinter LaserPrinter 가 각각을 구현할때 둘다 @Bean 으로 만들려고 한다면 다음과 같은 문제에 직면하게 됩니다 

```

Description:

Field printer in com.example.demo.SpringCoreApplication required a single bean, but 2 were found:
	- inkjetPrinter: defined in file [C:\Users\kimdo\Documents\workspace-spring-tool-suite-4-4.17.0.RELEASE\Spring-Core\target\classes\com\example\demo\InkjetPrinter.class]
	- laserPrinter: defined in file [C:\Users\kimdo\Documents\workspace-spring-tool-suite-4-4.17.0.RELEASE\Spring-Core\target\classes\com\example\demo\LaserPrinter.class]

```
지금 printer 타입의 bean 이 2가지가 있어서 bean 을 만들 수 없다는 뜻입니다 그렇기 이때 필요한것이 @Primary 입니다 

```
@Component
@Primary
public class LaserPrinter  implements Printer {

    @Override
    public void print() {
        System.out.println("LaserPrinter");
    }
}

```
그래서 LaserPrinter @Primary 달게 되면 bean 을 선정하고 주입할때 우선순위를 차지하게 됩니다 그렇기 때문에 항상 LaserPrinter 가 bean 으로 만들어져서 주입을 받게 됩니다 
그래서 그래서 우선순위로 bean 을 주입받고 싶을때에는 @Primary 를 사용합니다 

## @Qualifier
```

public interface Printer {

    void print();
}

==============================

@Component
public class InkjetPrinter  implements Printer {

    @Override
    public void print() {

        System.out.println("InkjetPrinter");

    }
}

==============================


@Component
public class LaserPrinter  implements Printer {

    @Override
    public void print() {
        System.out.println("LaserPrinter");
    }
}

==============================

@Component("zebraPrint")
public class ZebraPrint implements Printer {

    @Override
    public void print() {
        System.out.println("ZebraPrint");
    }
}


```

이번엔 Printer 를 상속받는 ZebraPrint 를 선언하고 이떄 bean 이름을 지정할 수 있는데 이때는 zebraPrint 이름을 짓게 됩니다 

```
@SpringBootApplication
public class SpringCoreApplication implements ApplicationRunner {

	@Autowired
	private Printer printer;

	@Autowired
	@Qualifier("zebraPrint")
	private Printer printer2;



	public static void main(String[] args) {

		SpringApplication.run(SpringCoreApplication.class, args);

	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		printer.print();
		printer2.print();
	}
}


```
그리고 bean 을 @Autowired 할때 @Qualifier("zebraPrint") 에서 bean 이름을 찾을 수 있게 지정을 할 수 있으면 그러면 이제 printer2 는 같은 타입의 print 가 있다 할지라도 
bean 이름이 zebraPrint 인 bean 만 찾아서 주입을 하게 됩니다

그럼 실전에서는 어떻게 쓰이느냐 보통 다중 DB 를 생성할 때 사용하게 됩니다 

## 다중DB 선언해서 사용하기 




