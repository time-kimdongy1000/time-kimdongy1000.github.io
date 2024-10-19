---
title: Spring Ioc
author: kimdongy1000
date: 2023-03-09 20:17
categories: [Back-end, Spring - Core]
tags: [IoC]
math: true
mermaid: true
---

## IoC 컨테이너
IoC inversion of control 제어의 반전 여기서 말하는 제어는 객체의 생성 주도권을 의미하는데 이전에 우리가 어떤 객체를 만들었는지 생각을 해보자

```

Student student = new Student();

```
우리가 직접 main 메서드 안에 new라는 객체 생성 연산자를 이용해서 생성자를 호출하여 객체를 만들었다 하지만 이제 이러한 작업들을 전부 IoC 컨테이너에게 맡기게 된다

## IoC 컨테이너란 
애플리케이션의 컴포넌트를 생성하고 관리하며 각 컴포넌트들 간의 의존성을 해결하는 컨테이너를 말합니다
이 IoC 컨테이너의 역할은 다음과 같습니다

1. 객체의 생성 및 소멸 관리 즉 객체의 탄생과 소멸까지 모든  라이플 사이클에 관해서 관리 감독하는 주체입니다
2. 의존성 주입 특정 곳에서 객체가 필요로 할 때 직접 주입하는 게 아니라 Ioc 컨테이너를 통해서 주입을 하게 됩니다

이 외에도 조금 더 있지만 보통 개발자들은 1번 및 2번에 대해서 많이 IoC 컨테이너 활용을 하게 됩니다


## Bean
Spring에서 Ioc 컨테이너에 의해 관리되는 객체를 빈(Bean)이라고 합니다 Bean 은 Spring Ioc 컨테이너에 의해 인스턴스화, 조립 및 관리되는 객체입니다
예를 들어서 아래와 같은 new 연산자로 쓰인 객체는 Bean 이 될 수 없습니다 그 이유는 new라는 객체는 Ioc 컨테이너에 의해서 관리되는 객체가 아니기 때문입니다

```

Student student = new Student(); // 이렇게 new 연산자로 사용된 객체는 IoC 컨 테이프가 관리하는 대상이 아니기 때문에 Bean이라고 하지 않습니다

```
그럼 뭐가 Bean 이 된다는 것일까? 

## ApplicationContext
ApplicationContext는 Spring에서 제공하는 IoC 컨테이너 한 종류로 Bean의 인스턴스화, 조립을 담당합니다 사실 Bean 을 관리하는 최상단은 BeanFactory 최상단 인터페이스가 존재하지만
편의성은 떨어져서 확장 기능이 많이 추가된 ApplicationContext IoC 컨테이너를 사용하게 됩니다


## Ioc 컨테이너 동작 방식 
```

ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");

```

사실 ApplicationContext 도 인터페이스라서 이를 구현하는 구현체를 사용하게 됩니다 그 중한 가지가 ClassPathXmlApplicationContext 가 구현체가 되는 것이고 이 특징은
ClassPathXmlApplicationContext classPath 위치상의 xml 파일을 읽어서 그 내용을 Ioc 컨테이너로 넣는 작업을 진행을 하게 됩니다 그럼 이제 이 클래스가 읽을 수 있는 xml 파일을 정의를 하게 될 것입니다

## Bean 정의 xml 
프로젝트 또는 예전 spring 문서들을 보게 되면 xml에 복잡하게 무엇인가 적혀 있는 것을 본 적이 있을 것이다 이런 프로젝트는 bean 정의서를 전부 xml로 관리를 하고 있는 것이다
즉 xml에 정의를 하고 그 xml 파일을 Ioc 컨테이너가 읽게 해서 Bean 을 불러오고 쓰는 것입니다

## bean.xml
```

<?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
	  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	  xmlns:context="http://www.springframework.org/schema/context"
	  xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd">
		
	  	<bean id = "MySystemInfo" class = "com.cybb.main.MySystemInfo"></bean>

  </beans>
```

처음부터 복잡하게 가지 말고 간단한 xml 파일을 하나 만들어서 bean 을 정의를 해보겠습니다 xml 태그들이 여러 개 있지만 우리가 봐야 할 핵심 태크는
`<bean id = "MySystemInfo" class = "com.cybb.main.MySystemInfo"></bean>` 이 부분이 어떤 bean 을 만들 것이라는 명세서입니다

이 bean의 속성은 두드러지게 3가지를 꼽을 수 있습니다 (물론 더 존재합니다)

* id 이 bean의 유일한 식별자이며 bean 이 생성될 이름을 뜻합니다
* class 빈으로 사용할 클래스의 경로를 지정합니다
* scope 여기서는 정의되지 않았지만 scope를 정의해서 bean 범위를 지정하게 되는데 (없으면 default singleton으로 지정)  거의 대부분의 애플리케이션은 싱글톤 범위에서 만들어지게 됩니다


## MySystemInfo

MySystemInfo는 이제 bean 될 클래스입니다 즉 우리는 생성자에 `System.out.println("MySystemInfo Bean 생성");` 걸어두었는데 bean 생성이 될 때도 생성자를 호출하게 됨으로  
우리는 이를 통해서 bean 이 생성된 것을 알 수 있을 것입니다

```
public class MySystemInfo {
	
	public MySystemInfo() {
		System.out.println("MySystemInfo Bean 생성");
	}	
}
```

## Bean 주입 및 기동 
```
@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner{

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
		MySystemInfo info =  ctx.getBean("MySystemInfo" , MySystemInfo.class);
	}
}

```
이때 ApplicationRunner는 springboot 가 완벽히 기동이 된 다음 제일 처음 호출되는 callback 함수입니다 그렇기에 우리는 이곳에 소스를 넣고 진행을 하겠습니다

`ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");`로 인해서 classPath 위치상의 bean.xml 을 읽으려고 하는 것입니다 이를 통해서 Ioc 컨테이너 생성하고 Bean 을 Ioc 컨테이너가 관리하게 만들 것입니다

`MySystemInfo info = ctx.getBean("MySystemInfo" , MySystemInfo.class);` 을 통해서 ctx의 getBean이라는 메서드를 통해서 Bean 을 가져와서 MySystemInfo 탑의 객체를 주입합니다 그럼 이 코드의 결과는 MySystemInfo 생성자의 즉시 호출이 될 것인데 결과를 보면

## 호출결과 
```
MySystemInfo Bean 생성
```

우리는 new 연산자를 사용하지 않고 오롯이 IoC 컨테이너가 제공하는 방식으로 객체를 넘겨받아서 생성자를 호출한 모습을 볼 수 있습니다 그럼 이제까지만 보면
IoC 컨테이너가 상당히 불편하고 어렵게 느껴질 것입니다 우리는 객체 하나를 만들 때 new 연산자 사용해서 객체를 만든 반면 Ioc 컨테이너가 제공하는 bean으로 객체를 쓰기 위해서 파일을 몇 개나 만들고 소스를 더욱더 길게 사용했습니다 그럼에도 이 Ioc를 써야 할 이유를 장점과 단점을 구분해서 보이도록 하겠습니다


## IoC 장점 
1. 느슨한 결합
객체 간의 의존성을 줄여주며 클래스들 간의 결합이 낮아지면서 새로운 기능을 추가하거나 기존 기능을 변경의 유연성 제공

2. 코드의 재사용성
IoC는 객체의 생성, 관리, 소멸과 같은 일반적인 작업들을 프레임워크가 처리하므로 개발자는 이러한 부분에 집중하지 않고 핵심 비즈니스 로직에 더 많은 시간을 할애할 수 있습니다.

## IoC 단점
1. 많은 양의 학습량
지금도 보았다 싶이 객체 하나를 만들어내기까지 수많은 클래스 그리고, 소스 들을 접했습니다 그래서 IoC에 관한 내용에 처음 접한 개발자는 이에 대한 공부를 진행을 해야 합니다

2. 코드의 추상화
우리는 앞에서 객체를 만들 때 new라는 연산자를 사용해서 객체를 만들어서 이 코드가 확실히 객체 생성 코드 인지 파악을 했는데 지금은 추상화가 많이 되어 있어서 이 코드의 역할이 정확히 무엇인지 파악하기 힘듭니다 이는 1번의 학습량과 관련이 있습니다


## JDBC VS MyBatis 
솔직히 이 부분으로 Ioc 컨테이너의 장점을 부각할 수 있는 비교 소스가 될 거 같습니다

## JDBC
```
public static void main(String[] args) {

	String db_url = "jdbc:oracle:thin:@localhost:1521:orcl";
	String user = "SCOTT";
	String password = "tiger";

	Connection conn = null;
	Statement stmt = null;
	ResultSet rs = null;
	HashMap<String, String> data = new HashMap<>();

	for(int a = 1; a < 46; a++) {
		String query ="SELECT COUNT(*) FROM lotto WHERE FIRSTNUM ="+a;
		try {
			conn = DriverManager.getConnection(db_url,user,password);
			stmt = conn.createStatement();
			rs = stmt.executeQuery(query);
			if(rs.next()) {
				System.out.println(a+" 개수는: "+rs.getInt(1));
				data.put(Integer.toString(a) , Integer.toString(rs.getInt(1)));
			}
		}catch (SQLException e) {
			e.printStackTrace();
		}finally {
			try {
				rs.close();
				stmt.close();
				conn.close();
				Thread thread = new Thread();
				thread.sleep(1000);
			}catch(Exception e) {
				e.printStackTrace();
			}
		}	
	}
}

```

이 소스는 예전 로또 프로그램 작성할 때 소스 일부분을 가져온 것입니다 지금 보면 DB 연결을 위해서 상단에는 DB 접속 정보를 등록해두었고
하단에는 쿼리를 포함한 실제 비즈니스 소스가 있습니다 사실 여기서 하고자 하는 것은 `SELECT COUNT(*) FROM lotto WHERE FIRSTNUM` 문장을 통해서
결과를 가져오는 게 주 목적인 프로그램인데 그 외적인 내용 예를 들어 DB 연결 정보 및 DB 연결 과정 이 적혀 있는 모습을 볼 수 있습니다

그리고 이런 JDBC의 가장 문제점은 다른 비즈니스 로직마다 새로운 연결을 열어서 진행을 해야 한다는 점입니다
즉 conn = DriverManager.getConnection(db_url, user, password); , stmt = conn.createStatement(); , rs = stmt.executeQuery(query); 반복적으로 말이죠
물론 연결 같은 것은 최상단 부모 클래스에서 진행을 한 뒤 공통으로 쓰게끔 만들어도 되지만 그 외에는 구현을 하는 소스 안에서 반복적으로 작업을 해주어야 합니다

그리고 역시나 끝이 나면 자원 반납

## mybatis 
물론 mybatis를 사용하기 위해서 사전에 db와 관련한 bean 들을 정의를 해주어야 합니다

```
@Configuration
@MapperScan(basePackages = "com.com.springBoot.dao") //mapper 스캔
public class DataBaseConfig {
	
	@Bean
	public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception{
		
		SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
		bean.setDataSource(dataSource);
		bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/static/mapper/*.xml")); // 쿼리를 담당할 xml 의 위치를 명시해둠
		bean.setConfigLocation(new PathMatchingResourcePatternResolver().getResource("classpath:/static/config/db/mybatis-config.xml")); // 다른 오라클 설정시 사용하는 config.xml 파일
			
		return bean.getObject();
		
	}
	
	@Bean
	public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
		return new SqlSessionTemplate(sqlSessionFactory);
	}
}

```
이런 bean 정의서를 만들고 

```
@Repository
@Mapper  
public interface RegisterMapper {
	
	public void SignUpRegisterUser(RegisterData registerData) throws Exception;

}
```
이 부분이 실제 쿼리를 호출하기 위한 비즈니스 로직이 되는 것이고

```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.com.springBoot.dao.RegisterMapper">

	<insert id="SignUpRegisterUser" parameterType="registerData">
		INSERT INTO WhatBookUser(
			USEREMAIL,
			USERPASSWORD,
			USERNICKNAME,
			USEREXPIRE,
			REGISTERDATE
		)VALUES(
			#{userEmail},
			#{userPassWord},
			#{userNickName},
			'Y',
			sysdate
		)
	</insert>	
</mapper>

```
위의 repository에 맞닿아 있는 mapper에 매핑이 되어서 쿼리가 실행되는 형식입니다 Ioc의 특징은 초기 설정은 좀 불편하고 어려울지는 몰라도 한번 설정을 해두면
개발자가 다시는 DB 연결이다든지, 차후 db 연결을 끊어준다는 이런 로직은 생각할 필요는 없습니다 그런 작업은 이제 Ioc 컨테이너에서 작업을 진행의 하기 때문이죠
이 때문에 개발자는 완전히 비즈니스 로직에만 신경을 쓰고 개발을 진행을 해 나가면 된다는 말씀을 드린 것입니다