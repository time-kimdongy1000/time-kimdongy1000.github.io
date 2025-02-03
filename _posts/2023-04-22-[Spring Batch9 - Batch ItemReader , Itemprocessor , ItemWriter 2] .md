---
title: Spring Batch9 - Batch ItemReader , Itemprocessor , ItemWriter
author: kimdongy1000
date: 2023-04-22 20:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## 지난시간 리뷰
지난시간에 우리는 간단한 쇼핑몰을 만들고 사용자들이 구매한 구매데이터를 정산 데이터로 옮긴 작업을 진행을 하였습니다 앞의 포스터에서는 jpa 를 활용해서 진행을 했는데 이번에는 
myBatis 를 활용해서 진행을 하겠습니다 DB는 mysql 로 진행을 하도록 하겠습니다

## 전체소스
<https://gitlab.com/kimdongy1000/spring-batch/-/commit/6278b883df88081d9cf66e115d75d62901611843>

JPA 예제와 같이 올라갔습니다 패키지 이름은 mybatis가 오늘 예졔이고 shpping 가 JPA 예제입니다 


## 의존성

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

## DB config 설정

```
@Configuration
public class DBConfig {

    @Value("${spring.datasource.url}")
    private String url;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
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
이는 DB Bean 을 생성하기 위한 config 파일입니다 

## 전체 Batch 소스 
```
@Configuration
@RequiredArgsConstructor
public class MyBatisBatchJob {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private SqlSessionFactory sqlSessionFactory;



    @Bean
    public Job mybatisJob(Step myBatisStep){

        return jobBuilderFactory.get("mybatisJob")
                .incrementer(new RunIdIncrementer())
                .start(myBatisStep)
                .build();
    }

    @Bean
    @JobScope
    public Step myBatisStep(ItemReader<Purchases> mybatisPurchasesReader , ItemProcessor<Purchases , Calculates> mybatisConvertData , ItemWriter<Calculates> calculatesItemWriter){

        return stepBuilderFactory.get("myBatisStep")
                .<Purchases , Calculates>chunk(100)
                .reader(mybatisPurchasesReader)
                .processor(mybatisConvertData)
                .writer(calculatesItemWriter)
                .build();
    }


    /*
    * ItemReader 이때 MyBatisPagingItemReaderBuilder 를 사용해서 데이터를 가져올것이고 
    * MyBatisPagingItemReaderBuilder 를 사용해서 불러옵니다 
    * 
    * sqlSessionFactory 는 우리가 Batch 를 사용할려고 하는 sqlSessionBean 을 넣어줍니다 
    * 
    * queryId 는 xml 에 매핑이 되는 id 로 원천 데이터를 불러올 쿼리문입니다 
    * 
    * pageSize 는 chunk 와 마찬가지로 한번에 처리할 값을 적어둡니다 
    * 
    * */
    @Bean
    @StepScope
    public ItemReader<Purchases> mybatisPurchasesReader(){

        return new MyBatisPagingItemReaderBuilder<Purchases>()
                .sqlSessionFactory(sqlSessionFactory)
                .queryId("spring.batch.mybatis.batch.select_batch_purchases")
                .pageSize(100)
                .build();
    }

    /*
    * 
    * ItemProcessor 우리는 JPA 에서 처리한것과 마찬가지로 데이터를 불러와서 가공을 한뒤에 처리할 예정입니다 
    * 
    * repository 를 따로 만들지 말고 위에서 정의한 sqlSessionFactory 에서 동일한 session 을 가져온뒤 xml 에 매핑이 되는 namespace 를 적어줍니다 
    * 이렇게 하는 이유는 아래서 설명을 드리겠습니다 
    * 
    * 
    * 
    * 
    * */
    @Bean
    @StepScope
    public ItemProcessor<Purchases , Calculates> mybatisConvertData(){

        final String todayFormat = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));

        return new ItemProcessor<Purchases, Calculates>() {
            @Override
            public Calculates process(Purchases item) throws Exception {



                int userId = item.getUserId();
                int itemId = item.getItemId();

                //Items items = itemRepository.findByItemId(itemId);
                //Users users = userRepository.findByUserId(userId);

                Items items = sqlSessionFactory.openSession().selectOne("com.example.demo.batch.mybatis.repository.ItemRepository.findByItemId" , itemId);
                Users users = sqlSessionFactory.openSession().selectOne("com.example.demo.batch.mybatis.repository.UserRepository.findByItemId" , userId);


                Calculates calculates = new Calculates(todayFormat + "-" + users.getUserID() + "-" + items.getItemName() + "-" + UUID.randomUUID()
                        ,users.getUserID()
                        ,items.getItemId()
                        ,item.getItemId()
                        ,users.getUserName()
                        ,items.getItemName()
                        ,items.getItemType()
                        ,items.getItemPrice()
                        ,item.getPurchaseIdDts());

                return calculates;
            }
        };

    }

    /*
    * 
    * 받은 데이터를 insert 하는 문구입니다 
    * ItemReader 하고는 조금 다르게 MyBatisBatchItemWriterBuilder 사용합니다 
    * 
    * sqlSessionFactory 사용할 DB 연결 객체를 넣어주시고
    * 
    * statementId insert 할 쿼리문을 만들어줍니다 
    * 
    * */
    @Bean
    public ItemWriter<Calculates> mybatisItemWriter(){

        return new MyBatisBatchItemWriterBuilder<Calculates>()
                .sqlSessionFactory(sqlSessionFactory)
                .statementId("spring.batch.mybatis.batch.insert_Calculates")
                .build();
    }




}


```
전체 소스입니다 이 소스에서는 주석을 달아놓았고 JPA 와 거의 동일한 패턴으로 가기때문에 이해 하는데는 무리가 없을 것입니다 다만 JPA 가 사용하는 구현체만 다를 뿐입니다 

## select_batch_purchases
```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="spring.batch.mybatis.batch">

    <select id="select_batch_purchases" parameterType="Map" resultType="com.example.demo.batch.mybatis.dao.Purchases">
        SELECT 	PurchaseId      AS purchaseId,
                UserID          AS userID,
                ItemId 		    AS	itemId,
                PurchaseId	    AS	purchaseId,
                PurchaseIdDts   AS purchaseIdDts
        FROM Purchases
        LIMIT #{_skiprows}, #{_pagesize}
    </select>

    <insert id = "insert_Calculates" parameterType="com.example.demo.batch.mybatis.dao.Calculates">
        INSERT INTO Calculates(
            CalculateId,
            UserId,
            UserName,
            ItemId,
            ItemName,
            ItemType,
            ItemPrice,
            PurchaseId,
            PurchaseIdDts

        ) values (
            #{calculateId},
            #{userId},
            #{userName},
            #{ItemId},
            #{itemName},
            #{itemType},
            #{itemPrice},
            #{purchaseId},
            #{purchaseIdDts}
        )

    </insert>
</mapper>


```
이 xml 에서는 ItemReader 에서 사용할 쿼리문과 ItemWriter 에서 사용한 쿼리문을 같이 놓았습니다

## item.xml 

```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.demo.batch.mybatis.repository.ItemRepository">

    <select id = "findByItemId" parameterType="Integer" resultType="com.example.demo.batch.mybatis.dao.Items">
        SELECT 	itemId      AS itemId,
                ItemName    AS itemName,
                ItemType    AS itemType,
                ItemPrice   AS itemPrice
        FROM Items
        WHERE itemId = #{item_id}

    </select>


</mapper>


```

## user.xml
```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.demo.batch.mybatis.repository.UserRepository">

    <select id = "findByItemId" parameterType="Integer" resultType="com.example.demo.batch.mybatis.dao.Users">

        SELECT 	UserID    AS userId,
                UserName  AS userName
        FROM Users
        WHERE UserID = #{user_id}

    </select>


</mapper>


```

우리는 ItemProcessor 에서 사용할 쿼리문을 따로 분리를 해두었습니다 이렇게 하면 JPA 에 동일한 동작을 하는 myBatis로 Batch 를 연동했습니다 다만 JPA 보다는 약간의 파일이 더 만들어진것을 볼 수 있습니다 아무래도 연결이나 이런것들을 정의하고 쿼리문을 직접 둬야 하기 때문입니다 

## 번외 

```

//Items items = itemRepository.findByItemId(itemId);
//Users users = userRepository.findByUserId(userId);

Items items = sqlSessionFactory.openSession().selectOne("com.example.demo.batch.mybatis.repository.ItemRepository.findByItemId" , itemId);
Users users = sqlSessionFactory.openSession().selectOne("com.example.demo.batch.mybatis.repository.UserRepository.findByItemId" , userId);

```

ItemProcessor 중간에 보면 주석잡힌 두개의 문구가 있습니다 위에 두개는 특별한 설정을 하지 않으면 오류가 발생합니다 오류는 
Cannot change the ExecutorType when there is an existing transaction 즉 트랜잭션이 일어나는 동안에는 추출타입을 변경할 수 없다는 뜻입니다 이 말이 무엇이냐 

우리는 현재 배치를 사용하면서 SqlSessionType 이 고정적으로 batchType 으로 설정이 되어 있습니다 그런데 우리가 
itemRepository.findByItemId(itemId); 문구를 호출하면 mybatis 일반 simple 타입으로 ExecutorType 변경할려고 합니다 그런데 지금은 트랜잭션이 동작중입니다 batchType 말이죠  
select -> insert 가 서로다른 쓰레드에서 일어나고 있으므로 이를 변경할 수 없다는 뜻입니다 그래서 오류가 발생하게 됩니다 이 경우에는 그냥 세션을 열어서 하나 사용하는게 제일 빠른 방법중 하나입니다 

오늘은 spring - batch 의 mybatis 에 대해서 알아보았습니다