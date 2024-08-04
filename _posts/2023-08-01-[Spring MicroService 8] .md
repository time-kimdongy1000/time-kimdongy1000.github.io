---
title: Spring MicroService 8 Spring MicroService Cloud-Config @RefreshScope
author: kimdongy1000
date: 2023-08-01 13:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

## @RefreshScope
오늘은 클라우드 config 에서 그중에서 특히 Client 에서 사용하는 애노테이션 @RefreshScope 에 대해서 알아볼려고 합니다
@RefreshScope는 Spring Cloud에서 사용하는 어노테이션으로, Spring 애플리케이션에서 구성 프로퍼티의 동적인 업데이트를 지원하는 데 사용됩니다. 
그럼 예제를 보면서 진행을 하겠습니다 마찬가지로 서버와 클라이언트로 나뉘고 도커를 활용할 예정입니다

## config-server maven
```

<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>2.7.1</version>

<properties>
  <java.version>11</java.version>
  <spring-cloud.version>2021.0.8</spring-cloud.version>
</properties>



<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-server</artifactId>
</dependency>


<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

```

서버는 이렇게 관리하겠습니다 그리고 마찬가지로 프러퍼티를 총 3개 application.properties , dev , prod 로 나누겠습니다

## config-server application.yml

```
spring:
  application:
    name: config-server

  profiles:
    active: native

  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config
server:
  port: 8087

```
이번에도 마찬가지로 읽어올 config 는 classpath 에 관리하도록 하겠습니다


## config-server.yml
```
spring:
  datasource:
    hikari:
      connection-timeout: 30000
      minimum-idle: 5
      maximum-pool-size : 20
      idle-timeout: 300000
      max-lifetime: 1800000
      auto-commit : true
      pool-name : MyHikariCP
      validation-timeout : 5000
      leak-detection-threshold : 2000
      connection-test-query : SELECT 1
 
```

## config-server-dev.yml
```
spring:
  datasource:
    hikari:
      url : jdbc:mysql://localhost:3308/dev_database
      username : dev_user
      password : dev_password
      driver-class-name : com.mysql.cj.jdbc.Driver



  config :
    value1 : test3
    value2 : test2

```

## config-server-prod.yml
```
spring:
  datasource:
    hikari:
      url : jdbc:mysql://localhost:3306/prod_database
      username : prod_user
      password : prod_password
      driver-class-name : com.mysql.cj.jdbc.Driver
```

이렇게 만들겠습니다 그리고

## config-server Dockerfile
```
FROM maven:3.8.4-openjdk-11 AS build

WORKDIR /app

COPY . .

RUN mvn package -DskipTests

FROM openjdk:11-jdk-slim

COPY --from=build /app/target/*.jar /config-server.jar

ENTRYPOINT ["java","-jar","/config-server.jar"]

```

## config-server docker-compose.yml
```
version: '3.8'

services:
    dev_db:
      image: mysql:latest
      container_name: dev_db
      environment:
        MYSQL_ROOT_PASSWORD: root_password
        MYSQL_DATABASE: dev_database
        MYSQL_USER: dev_user
        MYSQL_PASSWORD: dev_password
      ports:
        - "3308:3306"
      volumes:
        - dev_db_data:/var/lib/mysql

    prod_db:
      image: mysql:latest
      container_name: prod_db
      environment:
        MYSQL_ROOT_PASSWORD: root_password
        MYSQL_DATABASE: prod_database
        MYSQL_USER: prod_user
        MYSQL_PASSWORD: prod_password
      ports:
        - "3307:3306"
      volumes:
        - prod_db_data:/var/lib/mysql

    app:
      build: .
      container_name: app
      ports:
        - "8087:8087"

volumes:
  dev_db_data:
  prod_db_data:

```
스크립트는 간단하게 어플리케이션과 2개의 DB 를 만들어줍니다 이때 app 에 의존하지는 않게 만들었습니다 오늘 예제는 그것이 필요 없이 그냥 단순히 같이 기동만 되면 되기 때문입니다

## config-client maven
```


<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>2.7.1</version>


<properties>
  <java.version>11</java.version>
  <spring-cloud.version>2021.0.8</spring-cloud.version>
</properties>



<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-client</artifactId>
</dependency>


<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.18.32</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>


<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>8.0.29</version>
</dependency>

<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>2.2.2</version>
</dependency>


<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>


<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <version>2.2.224</version>
  <scope>runtime</scope>
</dependency>


<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

```

클라이언트 해당하는것들은 값을 많이 넣어줍니다 클라이언트에서는 DB 연결을 포함한 서버의 config 정보가 변경되면 감지해서 reload 할 `spring-cloud-starter-bootstrap` , `spring-boot-starter-actuator` 를 의존해서 사용할것입니다

## config-client application.yml
```

server:
  port: 8081

spring:

  application:
    name: config-client

  config:
    import: "optional:configserver:http://localhost:8087/"

  cloud:
    config:
      name: config-server
      profile: dev

  profiles:
    active: dev

  datasource:
    hikari:
      url: jdbc:h2:mem:testdb
      driver-class-name: org.h2.Driver
      username: sa
      password: password
      connection-timeout: 30000
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 300000
      max-lifetime: 1800000
      auto-commit: true
      pool-name: MyHikariCP
      validation-timeout: 5000
      leak-detection-threshold: 2000
      connection-test-query: SELECT 1

```
여기 보면 지난 예제에서 config 는 동일하지만 밑에보면 datasource 가 있습니다 이는 DB 가 없으면 임시 DB 연결을 위한 설정입니다 실제로 실행을 하게 되면 이 DB 를 보는 것이 아니라 config-server 에 있는 접속정보를 확인할것입니다


## config-client bootstrap.yml
```
spring:
  application:
    name: config-client
  config:
    import: "optional:configserver:http://localhost:8087/"
  cloud:
    config:
      name: config-server
      profile: dev
  profiles:
    active: dev
management:
  endpoints:
    web:
      exposure:
        include: refresh
```
그리고 bootstrap.yml 을 만들어서 진행을 하겠습니다 마찬가지로 이 설정정보는 conifg 정보를 분리하기 위함이지 이 정보를 application.yml 에 저장해서 사용해도 문제 없습니다

## config-client CustomValueConfig
```
@Configuration
@Getter
@Setter
@RefreshScope
@ConfigurationProperties(prefix = "spring.config")
public class CustomValueConfig {

    private String value1;
    private String value2;

}
```


## config-client MySqDataSourceConfig
```
@Configuration
@ConfigurationProperties(prefix = "spring.datasource.hikari")
@Getter
@Setter
@RefreshScope
public class MySqDataSourceConfig {


    private String url;


    private String username;


    private String password;



    private long connection_timeout;


    private int minimum_idle;


    private int maximum_pool_size;


    private long idle_timeout;


    private long max_lifetime;


    private boolean auto_commit;


    private String pool_name;


    private long validation_timeout;


    private long leak_detection_threshold;



    private String connection_test_query;

    @Bean("mysqlHikariConfig")
    @Primary
    public HikariConfig mysqlHikariConfig(){

        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(url);
        hikariConfig.setUsername(username);
        hikariConfig.setPassword(password);

        hikariConfig.setConnectionTimeout(connection_timeout);
        hikariConfig.setMinimumIdle(minimum_idle);
        hikariConfig.setMaximumPoolSize(maximum_pool_size);
        hikariConfig.setIdleTimeout(idle_timeout);
        hikariConfig.setMaxLifetime(max_lifetime);
        hikariConfig.setAutoCommit(auto_commit);
        hikariConfig.setPoolName(pool_name);
        hikariConfig.setValidationTimeout(validation_timeout);
        hikariConfig.setLeakDetectionThreshold(leak_detection_threshold);
        hikariConfig.setConnectionTestQuery(connection_test_query);

        return hikariConfig;

    }

    @Bean("mysqlDataSource")
    @Primary
    public DataSource mysqlDataSource(HikariConfig mysqlHikariConfig){
        DataSource mysqlDataSource = new HikariDataSource(mysqlHikariConfig);
        return mysqlDataSource;
    }

    @Bean("mysqlSessionFactory")
    @Primary
    public SqlSessionFactory mysqlSessionFactory(DataSource mysqlDataSource) throws Exception{

        SqlSessionFactoryBean mysqlSessionFactoryBean = new SqlSessionFactoryBean();
        mysqlSessionFactoryBean.setDataSource(mysqlDataSource);
        mysqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/static/mapper/mysql/*.xml"));

        return mysqlSessionFactoryBean.getObject();

    }

    @Bean("mysqlSessionTemplate")
    @Primary
    public SqlSessionTemplate mySqlsessionTemplate(SqlSessionFactory mySqlSessionFactory) throws  Exception{
        SqlSessionTemplate mysqlSessionTemplate = new SqlSessionTemplate(mySqlSessionFactory);
        return mysqlSessionTemplate;
    }

}

```
MySqDataSourceConfig 설정정보도 만들어줍니다 이에 대해서 지난시간에 마찬가지입니다



## config-client MySqlController
```
@RestController
public class MySqlController {

    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;

    @Autowired
    private CustomValueConfig customValueConfig;

    @GetMapping("/getMysql")
    public String getMysql(){

        int result = sqlSessionTemplate.selectOne(this.getClass().getName() + ".getMysql");

        return result + "" + customValueConfig.getValue1();
    }
}


```
간단한 controller 작성 후 이를 return 을 하겠습니다

그리고 config-server 와 config-client 를 기동하겠습니다 그리고 post-man 으로 요청을 하면 다음 요청을 받게 됩니다

![refresh_1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/95753989-a560-4305-9c75-82c6ff38457e)

이렇게 나오게 됩니다 이는 DB 의 결과 + config-server 의 정보가 합처저서 나오게 되는 것입니다 이때 config 서버의 설정을 바꾸고 docker 를 기동하겠습니다 이때 이때 config-client 는 변경된 정보를 다시 읽어야 함으로

http://localhost:8081/actuator/refresh 이렇게 요청을 주게 됩니다 그러면 변경된 정보를 감지해서 다시 이 값을 주입하게 됩니다 그리고 다시 요청을 주게 되면 이때 config-server 의 변경된 값으로 출력이 됩니다

## 그럼 DB 접속정보는 왜 넣어두었나?
사실 이 글의 목적은 동적DB 구성 예를 들어서 DB 가 변경이 되었을때도 다시 bean 을 생성해서 변경된 주소로 request 를 날리는지 확인을 했는데 DB 같은 접속정보는 Runtime 시 생성되어서 세팅이 되기 때문에 client-config 를 재기동하지 않는 이상 변경된 정보를 확인할 수 없습니다 다만 변경된 접속정보를 가져올 수 있으므로 동적으로 Connenct 객체를 만들어서 사용하면 이때는 또 변경된 주소로 사용이 가능합니다

## 정리

1. @RefreshScope 는 client 에서 server 에 변경된 config 정보를 읽어서 다시 세팅할 수 있는 애노테이션입니다

2. 클라이언트는 변경된 정보를 확인할려면 post 요청으로 http://localhost:8081/actuator/refresh 넣게 됩니다 그러면 response 로 변경된 정보의 key 가 넘어오게 됩니다

3. 런타임시 완성이 되는 Bean 의 설정정보는 변경하지 못하고(DataSource) 단순 String 같은 경우에서는 변경된 값을 바로 확인할 수 있습니다 
