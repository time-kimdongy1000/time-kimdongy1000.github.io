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
실전에서는 주로 다중DB 에서 많이 사용됩니다 예를 들어서 오라클 하고 mysql 을 동시에 연결할것이다 이때 주연결 테이블은 오라클이 될것이다 보조 연결 DB 는 mysql 이 될것입니다 

spring - boot  2.7.1
jdk11
oracle19c
mysql8 

이렇게 연결할 예정입니다


MAVEN 은 아래와 같다 
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
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>

<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>21.6.0.0.1</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.31</version>
</dependency>
```

적절한 의존성을 삽입해준다음 

## application.properties
```
# Oracle db
oracle.datasource.url=jdbc:oracle:thin:@192.168.46.129:1521:ORCL
oracle.datasource.username=kimdongy1000
oracle.datasource.password=***********

# Mysql db
mysql.datasource.url=jdbc:mysql://192.168.46.129:3306/kimdongy1000?characterEncoding=UTF-8&serverTimezone=UTC
mysql.datasource.username=kimdongy1000
mysql.datasource.password=************

```

먼저 application.properties 에 접속정보를 입력합니다 히카리CP 는 여러가지 설정을 커스텀 할 수 있지만 반드시 필요한 url , username , password 만 적어서 진행을 하겠습니다 

## OracleConfig

```

@Configuration
public class OracleConfig {

    @Value("${oracle.datasource.url}")
    private String url;

    @Value("${oracle.datasource.username}")
    private String username;

    @Value("${oracle.datasource.password}")
    private String password;

    @Bean
    @Primary
    public HikariConfig oracleHikariConfig(){

        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(url);
        hikariConfig.setUsername(username);
        hikariConfig.setPassword(password);

        return hikariConfig;

    }

    @Bean
    @Primary
    public DataSource oracleDataSource(HikariConfig oracleHikariConfig){
        DataSource oracleDataSource = new HikariDataSource(oracleHikariConfig);
        return oracleDataSource;
    }

    @Bean
    @Primary
    public SqlSessionFactory oracleSqlSessionFactory(DataSource oracleDataSource) throws Exception{

        SqlSessionFactoryBean oracleSqlSessionFactoryBean = new SqlSessionFactoryBean();
        oracleSqlSessionFactoryBean.setDataSource(oracleDataSource);
        oracleSqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/static/mapper/multiple/oracle/*.xml"));

        return oracleSqlSessionFactoryBean.getObject();

    }

    @Bean
    @Primary
    public SqlSessionTemplate oracleSqlSessionTemplate(SqlSessionFactory oracleSessionFactory) throws Exception{
        SqlSessionTemplate oracleSqlSessionTemplate = new SqlSessionTemplate(oracleSessionFactory);
        return oracleSqlSessionTemplate;
    }
}
```
내용은 별것이 없습니다 다만 Bean 들은 전부 @Primary 붙었습니다 주 테이블이 오라클DB 로 생각을 하기 때문에 런타임시에는 전부 오라클 DB 를 우선으로 bean 생성하고 주입합니다 

## MysqlConfig
```
@Configuration
public class MysqlConfig {

    @Value("${mysql.datasource.url}")
    private String url;

    @Value("${mysql.datasource.username}")
    private String username;

    @Value("${mysql.datasource.password}")
    private String password;

    @Bean("mysqlHikariConfig")
    public HikariConfig mysqlHikariConfig(){

        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(url);
        hikariConfig.setUsername(username);
        hikariConfig.setPassword(password);

        return hikariConfig;

    }

    @Bean("mysqlDataSource")
    public DataSource mysqlDataSource(@Qualifier(value = "mysqlHikariConfig") HikariConfig mysqlHikariConfig){
        DataSource mysqlDataSource = new HikariDataSource(mysqlHikariConfig);
        return mysqlDataSource;
    }

    @Bean("mysqlSessionFactory")
    public SqlSessionFactory mysqlSessionFactory(@Qualifier(value = "mysqlDataSource") DataSource mysqlDataSource) throws Exception{

        SqlSessionFactoryBean mysqlSessionFactoryBean = new SqlSessionFactoryBean();
        mysqlSessionFactoryBean.setDataSource(mysqlDataSource);
        mysqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:/static/mapper/multiple/mysql/*.xml"));

        return mysqlSessionFactoryBean.getObject();

    }

    @Bean("mysqlSessionTemplate")
    public SqlSessionTemplate mySqlsessionTemplate(@Qualifier(value = "mysqlSessionFactory") SqlSessionFactory mySqlSessionFactory) throws  Exception{
        SqlSessionTemplate mysqlSessionTemplate = new SqlSessionTemplate(mySqlSessionFactory);
        return mysqlSessionTemplate;
    }

}
```
같은 내용이긴 한데 Bean 전부 이름이 들어가 있는것을 볼 수 있고 주입할 대상들도 전부 @Qualifier 로 해서 이때 value 를 지정을 하면 해당 bean 을 찾게 됩니다 즉 

```

@Bean("mysqlHikariConfig")
public HikariConfig mysqlHikariConfig(){

    HikariConfig hikariConfig = new HikariConfig();
    hikariConfig.setJdbcUrl(url);
    hikariConfig.setUsername(username);
    hikariConfig.setPassword(password);

    return hikariConfig;

}

@Bean("mysqlDataSource")
public DataSource mysqlDataSource(@Qualifier(value = "mysqlHikariConfig") HikariConfig mysqlHikariConfig){
    DataSource mysqlDataSource = new HikariDataSource(mysqlHikariConfig);
    return mysqlDataSource;
}

```
이 두 줄만 살펴보면 제일먼저 HikariConfig bean 을 만들때 이름으로 mysqlHikariConfig 되면 런타임시 이 bean 이 이름으로 만들게 됩니다 즉 타입상관 없이 이름이 중복되지 않으면
같은타입을 생성할 수 있습니다 

그리고 주입할 대상에는 반드시 @Qualifier(value = "mysqlHikariConfig") 이렇게 value 에 해당 이름을 찾아서 주입을 하게 됩니다 그러면 다른 이름의 bean 이 주입이 되는것이 아니라 
반드시 같은 이름으로 생성된 bean 을 찾아서 주입을 하게 됩니다 

그렇기 때문에 OracleConfig 와 MysqlConfig 는 타입은 같지만 결국 bean 이름이 다르기 때문에 어플리케이션 런타임시 충돌나지 않습니다 

## OracleRepository 

```
public class OracleRepository {

    @Autowired
    private SqlSessionTemplate oracleSessionTemplate;


    public int oracleResult() {

        return oracleSessionTemplate.selectOne(this.getClass().getName() + ".test_oracle");
    }
}

```
그리고 쿼리를 불러올떄는 SqlSessionTemplate 을 불러와서 호출하게 되는데 이때 @Autowired 아무것도 적지 않으면 기본적으로 @Primary 타입으로 된 bean 을 찾아서 주입을 하게 됩니다 

## MysqlRepository 

```
@Repository
public class MysqlRepository {

    @Autowired(required = false)
    @Qualifier(value = "mysqlSessionTemplate")
    public SqlSessionTemplate mySqlSessionTemplate;

    public int mysql_result(){

        return mySqlSessionTemplate.selectOne(this.getClass().getName() + ".test_mysql");
    }
}
```
반대로 mysql 쿼리를 불러올때는 주입 bean 의 이름을 정확히 찾아서 주입을 해야 함으로 @Autowired(required = false) 이 설정은 이 bean 이 있을 수도 있고 없을 수도 있으니 bean 이 없어도 의존 에러를 발생시키지 않습니다 @Qualifier(value = "mysqlSessionTemplate") 이름으로 bean 을 찾아서 주입하기 위한 장치입니다 

## oracle.xml 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.cybb.main.repository.OracleRepository">

    <select id="test_oracle" resultType="Integer">
        SELECT 1 + 2 + 3 + 4 + 5 FROM DUAL
    </select>
</mapper>


```
테이블 유무보다는 이렇게 dual 테이블로 연결을 해서 연산결과를 return 받을것이고 

## mysql.xml 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.cybb.main.repository.MysqlRepository">

    <select id="test_mysql" resultType="Integer">
        SELECT 1 + 2 + 3 + 4 + 5 + 6 + 7 + 8 + 9 + 10
    </select>

</mapper>
```
8.0 이상 버전 부터는 mysql 도 daul 테이블을 도입할 수 있는데 oracle 하고 차이점을 보기 위해서 일부로 쓰지 않았습니다 

## 쿼리 호출
```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {

	@Autowired
	private OracleRepository oracleRepository;

	@Autowired
	private MysqlRepository mysqlRepository;


	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		int oracle_result = oracleRepository.oracleResult();
		int mysql_result = mysqlRepository.mysql_result();;
		System.out.println("Connect_oracle :" + oracle_result);
		System.out.println("Connect_mysql :" + mysql_result);

	}
}

```
쿼리는 이렇게 각각 repository 를 호출하는것으로 적었고 결과는 

```
Connect_oracle :15
Connect_mysql :55

```

이렇게 나와서 다중DB 연결할떄  @Primary 역활과 @Qualifier 역활에 대해서 알아보았습니다 




