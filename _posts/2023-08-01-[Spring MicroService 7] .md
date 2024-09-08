---
title: Spring MicroService 7 Spring MicroService Cloud-Config 2
author: kimdongy1000
date: 2023-08-01 12:30
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  Cloud-Config]
math: true
mermaid: true
---

그럼 이번시간에는 지난시간에 작성한

config-server 를 가지고 이를 바인딩 하는 config-client 를 만들어보겠습니다 시나리오는 다음과 같습니다

1) config-server 는 MySql 연결에 필요한 연결 정보를 적어둡니다

2) 공통정보는 default , 개발 특화는 dev , 운영 특화는 prod 로 나눠서 만들어놓습니다

3) docker-compose 로 config-server 와 dev_db , prod_db 를 나눠서 컨테이너로 올립니다

4) config-client - 는 config-server 의 정보를 읽어서 mysql 의존성에 데이터를 바인딩합니다

이런 시나리오로 갈것입니다

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

```

의존성을 추려서 넣었습니다

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

앞의 예제와 마찬가지로 config 서버의 classpath 의 config 폴더 안의 데이터를 읽을 것입니다

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
      url : jdbc:mysql://localhost:3306/dev_database
      username : dev_user
      password : dev_password
      driver-class-name : com.mysql.cj.jdbc.Driver
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

파일명만 보면 알 수 있다 싶히 공통정보는 config-server.yml  에 넣어두고 개별적인 정보는 각각의 properties 에 두었습니다

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
        - "3306:3306"
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
docker-compose 는 이와 같이 작성을 합니다 그리고 `docker-compose up --build` 로 기동을 합니다 그러면 DockerContainer 에는 총 3개의 이미지가 실행이 될것입니다

![docker_compose_a](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/83d3a636-3e98-4e14-bd30-035ebee7402a)

그럼 이제 config-client 를 작성하겠습니다

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
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>


<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.32</version>
    <scope>provided</scope>
</dependency>

```

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

```

application.yml 은 이와 같이 구성을 하게 됩니다 중요한것은 config.import 와 cloud.config 의 아래 값들입니다 이 값들을 맞춰서 config-server 에 요청을 하게 됩니다

## config-client MySqDataSourceConfig

```
@Configuration
@ConfigurationProperties(prefix = "spring.datasource.hikari")
@Getter
@Setter
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
    public DataSource mysqlDataSource(HikariConfig mysqlHikariConfig){
        DataSource mysqlDataSource = new HikariDataSource(mysqlHikariConfig);
        return mysqlDataSource;
    }

    @Bean("mysqlSessionFactory")
    public SqlSessionFactory mysqlSessionFactory(DataSource mysqlDataSource) throws Exception{

        SqlSessionFactoryBean mysqlSessionFactoryBean = new SqlSessionFactoryBean();
        mysqlSessionFactoryBean.setDataSource(mysqlDataSource);
        mysqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/static/mapper/mysql/*.xml"));

        return mysqlSessionFactoryBean.getObject();

    }

    @Bean("mysqlSessionTemplate")
    public SqlSessionTemplate mySqlsessionTemplate(SqlSessionFactory mySqlSessionFactory) throws  Exception{
        SqlSessionTemplate mysqlSessionTemplate = new SqlSessionTemplate(mySqlSessionFactory);
        return mysqlSessionTemplate;
    }

}

```
DB 와 통신 할 수 있게 SqlSessionTemplate 을 구성합니다 이때 존재하는 모든 의존성은 서버의 데이터를 읽어서 런타임시 구성을 하게 됩니다

## config-client MySqlController

```
@RestController
public class MySqlController {

    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;

    @GetMapping("/getMysql")
    public String getMysql(){

        int result = sqlSessionTemplate.selectOne(this.getClass().getName() + ".getMysql");

        return result + "";
    }
}


```

간단하게 controller 을 작성후 쿼리 호출 그리고 이 값을 return 을 해주겠습니다

## config-client test.xml
```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.cybb.main.MySqlController">

    <select id="getMysql" resultType="Integer">

        SELECT 1 + 2 FROM DUAL
    </select>

    <select id="getMysql2" resultType="Integer">

        SELECT 3 + 4 FROM DUAL
    </select>

</mapper>

```

xml 은 간단하게 하겠습니다

그럼 docker 컨테이너를 가동한채로 config-client 를 내장 톰캣으로 기동을 하겠습니다 기동을 하면

## config-client log

ConfigServicePropertySourceLocator : Fetching config from server at : http://localhost:8888
Connect Timeout Exception on Url - http://localhost:8888. Will be trying the next url if available
Could not locate PropertySource: I/O error on GET request for "http://localhost:8888/config-server/dev": Connection refused: connect; nested exception is java.net.ConnectException: Connection refused: connect

로그를 보면 좀 특이한점이 있습니다 먼저 기본설정인 8888 포트로 요청을 날리게 됩니다 이쪽으로 연결이 안될때는 우리가 설정한 곳으로 연결을 하게 됩니다

The following 1 profile is active: "dev"
Fetching config from server at : http://localhost:8087/
Located environment: name=config-server, profiles=[dev], label=null, version=null, state=null

그 다음 현재 active 버전을 확인 후 이제 우리가 설정한 서버인 http://localhost:8087/ 로 config 호출을 하게 됩니다

이렇게 하면 client 프로그램을 주로 만질 개발자들은 중앙서버의 설정을 굳이 알 필요 없이 바로 DB 같은 설정정보를 불러와서 사용할 수 있습니다