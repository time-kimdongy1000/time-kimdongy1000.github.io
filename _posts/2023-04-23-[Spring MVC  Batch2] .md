---
title: Spring Batch2
author: kimdongy1000
date: 2023-04-23 10:45
categories: [Back-end, Spring - MVC]
tags: [ MVC , Bactch ]
math: true
mermaid: true
---

## Batch

지난시간에 우리는 batch 를 활용해서 batch 가 어떻게 사용되는지 왜 사용하는지에 대해서 공부해보았다 다만 대용량 데이터가 없다보니 효용성을 크게 느끼지 못해서 
그렇다고 한번에 큰 무엇인가를 만들기보다는 작은거 하나하나 새로 만드는것이 좋아보인다 그래서 이번시간에는 

A 테이블 데이터를 select 해와서 이름만 다른 B 테이블로 insert 하는것을 해보겠습니다 데이터는 1개부터 10개 까지 해보고 다음것으로 진행을 해보겠습니다 
그럼 세상 간단한 테이블을 먼저 만들도록 하겠습니다

```

CREATE TABLE EMP (

	name varchar2(20) , 
	emp_no varchar2(20) 
)

CREATE TABLE EMP2 (

	name varchar2(20) , 
	emp_no varchar2(20) 

)

```

우리는 EMP 테이블을 조회해서 EMP2 테이블로 옮겨보겠습니다. 

## 데이터 한개 옮기기 
처음엔 데이터 한개만 옮겨 보겠습니다 

INSERT INTO EMP (name , emp_no) VALUES ('emp1' , '20230613_emp1'); 여기 있는 데이터를 EMP2 테이블로 옮기겠습니다 다음과 같은 코드가 나올것입니다 
먼저 DB 를 설정해야 합니다 DB 가 기본이되는것이기에 DB 설정을 위한 application.properties 와 DataSourceConfig 를 설정하겠습니다 다만 우리가 MyBatis 를 설정하는것처럼 
SqlSessing Bean 을 설정하지는 않을 것입니다 


## maven 

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-batch</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.springframework.batch</groupId>
	<artifactId>spring-batch-test</artifactId>
	<scope>test</scope>
</dependency>

<dependency>
	<groupId>com.oracle.database.jdbc</groupId>
	<artifactId>ojdbc8</artifactId>
	<version>21.6.0.0.1</version>
</dependency>

```
메이븐은 간단하게 batch 를 쓸 수 있는 라이브러리와 ,  jdbc 를 연결할 수 있는 ojdbc를 준비합니다 spring-boot-starter-batch 안에 DB 연결할 수 있는 라이브러리 
Hikari , DBCP 등 다양하게 사용할 수 있지만 저는 가장 대중적이면서 가장 안정적으로 평가받는 Hikari 를 사용하겠습니다 


## DataSourceConfig.java 

```

package com.cybb.main.config;

@Configuration
public class DataSourceConfig {

    @Value("${spring.datasource.url}")
    public String url;
    @Value("${spring.datasource.username}")
    public String username;
    @Value("${spring.datasource.password}")
    public String password;

    @Bean
    public HikariDataSource hikariDataSource(){

        HikariDataSource hikariDataSource = new HikariDataSource();

        hikariDataSource.setJdbcUrl(url);
        hikariDataSource.setUsername(username);
        hikariDataSource.setPassword(password);

        return hikariDataSource;
    }

    @Bean
    public DataSource dataSource (){
        DataSource dataSource = new HikariDataSource(hikariDataSource());
        return dataSource;
    }
}

```

기본적으로 xml 을 통해서 통신을 할 이유가 없으므로 DataSource 만 정의해서 Bean 을 생성하겠습니다  

그러면 이제 emp 테이블 한개를 emp2 테이블로 옮겨보겠습니다 기존에 사용하는 용어는 동일합니다 

## BatchConfiguration

```
@Configuration
@EnableBatchProcessing
public class BatchConfiguration extends DefaultBatchConfigurer {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private DataSource dataSource;

    @Override
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public ItemReader<EmpDto> reader() {
        JdbcCursorItemReader<EmpDto> reader = new JdbcCursorItemReader<>();
        reader.setDataSource(dataSource);
        reader.setSql("SELECT name , emp_no FROM EMP");
        reader.setRowMapper(new BeanPropertyRowMapper<>(EmpDto.class));
        return reader;
    }

    @Bean
    public ItemProcessor<EmpDto, EmpDto> processor() {
        return new ItemProcessor<EmpDto, EmpDto>() {
            @Override
            public EmpDto process(EmpDto item) {
                // 데이터 가공 및 필터링 작업 수행
                return item;
            }
        };
    }

    @Bean
    public ItemWriter<EmpDto> writer() {
        JdbcBatchItemWriter<EmpDto> writer = new JdbcBatchItemWriter<>();
        writer.setDataSource(dataSource);
        writer.setSql("INSERT INTO EMP2 (name, emp_no) VALUES (:name, :emp_no)");
        writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
        return writer;
    }

    @Bean
    public Step myStep(ItemReader<EmpDto> reader, ItemProcessor<EmpDto, EmpDto> processor, ItemWriter<EmpDto> writer) {
        return stepBuilderFactory.get("myStep")
                .<EmpDto, EmpDto>chunk(1) // 청크 크기 설정
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job myJob(Step myStep) {
        return jobBuilderFactory.get("myJob")
                .start(myStep)
                .build();
    }
}


```
클래스 상단에 정의된 애노테이션은 아시다 싶이 batch 프로세서를 가동시키고 싶으면 사용하는 애노테이션이죠 
`@Configuration @EnableBatchProcessing` @configuration 은 이 클래스가 설정파일이라는 것을 알려주고 @EnableBatchProcessing 이 클래스 파일이 현재 
배치를 사용할 것이라는 것을 알려주고 실행시 자동으로 실행되게끔 유도합니다 

```

@Autowired
private DataSource dataSource;

@Override
public void setDataSource(DataSource dataSource) {
	this.dataSource = dataSource;
}

```

첫번째 batchJob 을 설정하는 필드 말고 이 필드가 보인다 이 필드는 우리는 이제 DB 에 연결을 해서 작업을 할 예정이기에 아까 DataSourceConfig 만들어진 Bean 을 DataSource 에 담아놓고 그것을 setDataSource 를 통해서 이 배치가 우리가 연결한 DB 를 사용하게끔 하는것입니다 

```

@Bean
public ItemReader<EmpDto> reader() {
	JdbcCursorItemReader<EmpDto> reader = new JdbcCursorItemReader<>();
	reader.setDataSource(dataSource);
	reader.setSql("SELECT name , emp_no FROM EMP");
	reader.setRowMapper(new BeanPropertyRowMapper<>(EmpDto.class));
	return reader;
}

```
ItemReader 최초 배치 실행시 원천이 되는 데이터를 읽어오는 소스로 
Step 의 첫번째 구성요소로 데이터를 읽어오는 역활을 합니다 주로 데이터소스에서 데이터를 읽어오는 역활을 수행합니다 

이때 우리는 앞에서 static 한 ArrayList 를 읽었지만 여기서는 DB 의 데이터를 읽을 것이므로 
`JdbcCursorItemReader<EmpDto> reader = new JdbcCursorItemReader<>();` 를 통해서 
DB 를 통해 진행을 할것이다 

`reader.setDataSource(dataSource);` 를 통해서 읽어야 할 db 를 알려주고 

```

reader.setSql("SELECT name , emp_no FROM EMP");
reader.setRowMapper(new BeanPropertyRowMapper<>(EmpDto.class));
return reader;

```

쿼리 문을 통해서 데이터를 읽어온뒤 BeanPropertyRowMapper 하나하나씩 EmpDto.class 맞게 세팅을 해주게 된다 

```
@Bean
public ItemProcessor<EmpDto, EmpDto> processor() {
	return new ItemProcessor<EmpDto, EmpDto>() {
		@Override
		public EmpDto process(EmpDto item) {
			// 데이터 가공 및 필터링 작업 수행
			return item;
		}
	};
}
```

우리는 여기서 부가적인 작업은 필요 없음으로 가공없이 넘도록 하겠습니다 

```

@Bean
public ItemWriter<EmpDto> writer() {
	JdbcBatchItemWriter<EmpDto> writer = new JdbcBatchItemWriter<>();
	writer.setDataSource(dataSource);
	writer.setSql("INSERT INTO EMP2 (name, emp_no) VALUES (:name, :emp_no)");
	writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
	return writer;
}

```

그리고 우리는 ItemReader ItemProcessor 통해서 읽어온 데이터와 가공된 데이터를 통해서 이제 DB 에 데이터를 쓰는것으로 가겠습니다 읽어올때는 `JdbcCursorItemReader<EmpDto> reader = new JdbcCursorItemReader<>();` 가 쓰였다면 쓰일때에는 `JdbcBatchItemWriter<EmpDto> writer = new JdbcBatchItemWriter<>();` 가 쓰일 예정입니다 

마찬가지로 writer.setDataSource(dataSource); 를 통해서 insert 할 DB 를 세팅을 하고 
이를 볼때 알 수 있는건 서로 다른 DB 또한 가능하다는것을 알 수 있습니다 이 예제 또한 만들어보고 진행을 하겠습니다 

```

writer.setSql("INSERT INTO EMP2 (name, emp_no) VALUES (:name, :emp_no)");
	writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());

```

그리고 마지막으로 쿼리를 통해서 insert 를 시켜줍니다 그러면 batch 는 데이터를 어디서 읽어오는것 부터 시작해서 데이터 가공 및 어디로 wirte 하는거 까지 전부 진행이 되었습니다 

여기서 마지막으로 우리는 step 들을 등록을 해주어야 합니다 그건 지난 포스트에도 적었지만 마찬가지로 

```

@Bean
public Step myStep(ItemReader<EmpDto> reader, ItemProcessor<EmpDto, EmpDto> processor, ItemWriter<EmpDto> writer) {
	return stepBuilderFactory.get("myStep")
			.<EmpDto, EmpDto>chunk(1) // 청크 크기 설정
			.reader(reader)
			.processor(processor)
			.writer(writer)
			.build();
}

@Bean
public Job myJob(Step myStep) {
	return jobBuilderFactory.get("myJob")
			.start(myStep)
			.build();
}

```

를 통해서 각각 정의하고 사용하게 됩니다 그리고 실행을 하게 되면 EMP2 테이블에 데이터 한개가 들어와 있음을 알 수 있습니다.









