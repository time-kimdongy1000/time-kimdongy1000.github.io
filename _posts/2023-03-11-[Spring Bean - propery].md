---
title: Spring Ioc 와 Bean2
author: kimdongy1000
date: 2023-03-12 11:22
categories: [Back-end, Spring - Core]
tags: [Bean]
math: true
mermaid: true
---

## setter 로 bean 생성
지난시간엔 생성자로 Bean 을 만들주고 주입하는 방식에 대해서 공부했는데 이번시간에는 setter 로 Bean 을 주입하는 방식으로 해보겠습니다 
좀 쉬운 예제를 가져올려고 했지만 우리 myBatis 를 활용해서 setter 로 bean 주입하는 방식에 대해서 진행을 해보겠습니다 


## 추가 maven
```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.31</version>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>

```
maven 은 다음처럼 더 추가를 해주겠습니다 

## mysqlBean.xml 
```

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd">

    <bean id = "hikariConfig" class = "com.zaxxer.hikari.HikariConfig">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
        <property name= "jdbcUrl" value = "jdbc:mysql://192.168.46.129:3306/kimdongy1000?characterEncoding=UTF-8&amp;serverTimezone=UTC" />
        <property name="username" value="kimdongy1000" />
        <property name="password" value="11111" />
    </bean>

    <bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
        <constructor-arg ref="hikariConfig" />
    </bean>


    <bean id = "sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <property name="mapperLocations" value="classpath:/static/mapper/mysql/*.xml"/>
    </bean>

    <bean id = "sqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
        <constructor-arg ref="sqlSessionFactory"/>
    </bean>

    <bean id = "mysqlRepository" class="com.example.demo.repository.MysqlRepository">
        <constructor-arg ref = "sqlSessionTemplate" />
    </bean>

</beans>

```

## SpringCoreApplication.java
```

package com.example.demo;

import com.example.demo.repository.MysqlRepository;
import org.mybatis.spring.SqlSessionTemplate;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import javax.sql.DataSource;
import java.sql.Connection;

@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
		public class SpringCoreApplication implements ApplicationRunner {

	public static void main(String[] args) {
		SpringApplication.run(SpringCoreApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		ApplicationContext ctx = new ClassPathXmlApplicationContext("mysqlBean.xml");

		MysqlRepository mysqlRepository = ctx.getBean("mysqlRepository" , MysqlRepository.class);
		System.out.println(mysqlRepository.result());
	}
}

```

## MysqlRepository.java

```

package com.example.demo.repository;

import org.mybatis.spring.SqlSessionTemplate;

public class MysqlRepository {

    private final SqlSessionTemplate sqlSessionTemplate;

    public MysqlRepository(SqlSessionTemplate sqlSessionTemplate){
        this.sqlSessionTemplate = sqlSessionTemplate;
    }

    public int result(){

        return sqlSessionTemplate.selectOne(this.getClass().getName() + ".test");
    }
}

```

## tes.xml 
```

<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- 맵핑될 DAO 인터페이스의 Full name 을 줍니다. -->
<mapper namespace="com.example.demo.repository.MysqlRepository">

    <select id="test" resultType="Integer">
        SELECT 1 + 2 + 3 + 4 + 5 FROM DUAL
    </select>

</mapper>
```
이렇게 총 4개의 파일을 만들어주고 bean 선언 파일만 보겠습니다 

```

<bean id = "hikariConfig" class = "com.zaxxer.hikari.HikariConfig">
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
    <property name= "jdbcUrl" value = "jdbc:mysql://192.168.46.129:3306/kimdongy1000?characterEncoding=UTF-8&amp;serverTimezone=UTC" />
    <property name="username" value="kimdongy1000" />
    <property name="password" value="11111" />
</bean>


```

이 파일을 보면 이전에는 생성자를 통한 bean 으로 사용할떄 `<constructor-arg ref="hikariConfig" />` 태그로 이와 같이 constructor-arg 를 사용해서 주입을 했지만 
setter 로 주입을 할때는 property 로 주입을 합니다 그럼 com.zaxxer.hikari.HikariConfig java 파일을 열어보면

```

public void setDriverClassName(String driverClassName)
public void setJdbcUrl(String jdbcUrl)
public void setUsername(String username)
public void setPassword(String password)


```

이렇게 총 4개의 setter 를 볼 수 있는데 bean 으로 주입할때는 property 태그를 활용해서 넣을 수 있습니다 그럼 이 bean 으로 hikariConfig 가 만들어지고 그렇게 
Bean 을 삽입해서 mysql 을 연결할 수 있는것입니다 

그렇게 해서 최종 결과를 살펴보면

```

@Override
public void run(ApplicationArguments args) throws Exception {

    ApplicationContext ctx = new ClassPathXmlApplicationContext("mysqlBean.xml");

    MysqlRepository mysqlRepository = ctx.getBean("mysqlRepository" , MysqlRepository.class);
    System.out.println(mysqlRepository.result());
}

```

우리가 mysqlBean 을 만들때 하단에 mysqlRepository bean 을 생성해서 그 bean 주입해주고 그 결과를 반환받을 쿼리 까지 사용한 것입니다 그래서 결과는 

```
2023-03-12 11:01:23.381  INFO 7028 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
15
```

이렇게 HikariPool-1 - Start completed. 연결하고 15라는 값이 세팅이 되는것을 확인할 수 있습니다 