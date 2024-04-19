---
title: Spring Batch7 - Batch ItemReader , Itemprocessor , ItemWriter
author: kimdongy1000
date: 2023-04-22 19:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## Tasklet VS ItemReader , Itemprocessor , ItemWriter
오늘부터는 ItemReader , ItemProcessor , ItemWriter 에 대해서 공부를 할 것입니다 앞전까지는 우리는 Tasklet 을 step 에 등록을 해서 job 을 진행을 시켰다면 이제부터는 데이터를 읽고 , 가공하고 , 다시 쓰는 방식인 ItemReader , ItemProcessor , ItemWriter 을 공부할것입니다 그 전에 우리는 사전작업을 할것인데 간단한 쇼핑몰 ERD 구축후 진행을 하겠습니다 

## ERD

![ERD](/assets/img/post_img/batch_2023-04-22_19_00_1.png)


ERD 간단하게 만들어줄것입니다 

## ERD 스크립트 (mysql)
CREATE TABLE Users (
 UserID        INT NOT NULL AUTO_INCREMENT,
 UserName     VARCHAR(100) ,
 PRIMARY KEY(UserID)
);

CREATE TABLE Items (
 ItemId        INT NOT NULL AUTO_INCREMENT,
 ItemName     VARCHAR(100) ,
 ItemType     VARCHAR(100) ,
 ItemPrice     INT,
 PRIMARY KEY(ItemId)
);

CREATE TABLE Purchases (
 PurchaseId     VARCHAR(100) NOT NULL,
 UserID     	INT,
 ItemId     	INT,
 PurchaseIdDts  DATE, 
 PRIMARY KEY(PurchaseId)
);


CREATE TABLE Calculates (
 CalculateId    varchar(100) NOT NULL ,
 UserId     	INT,
 UserName     	Varchar(100),
 ItemId  		INT, 
 ItemName  		Varchar(100),
 ItemType  		Varchar(100),
 ItemPrice  	INT,
 PurchaseId  	Varchar(100),
 PurchaseIdDts  DATE,
 
 PRIMARY KEY(CalculateId)
);

## 기초 데이터 스크립트 

```
INSERT INTO Users(UserName) values('KIM');
INSERT INTO Users(UserName) values('LEE');
INSERT INTO Users(UserName) values('TIME');
INSERT INTO Users(UserName) values('HEO');
INSERT INTO Users(UserName) values('BEAN');


INSERT INTO Items(ItemName , ItemType , ItemPrice) values('수박' 		, '과일' , '1000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('오렌지' 	, '과일' , '1500');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('청포도' 	, '과일' , '1700');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('포도' 		, '과일' , '2000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('딸기' 		, '과일' , '2300');

INSERT INTO Items(ItemName , ItemType , ItemPrice) values('java 100일 문법' 			, '책' , '5000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('oracle 100일 문법' 		, '책' , '6000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('spring 이주일만에  끝내기' 	, '책' , '13000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('보안이란 무엇인가' 			, '책' , '20000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('알고리즘 입문' 				, '책' , '50000');

INSERT INTO Items(ItemName , ItemType , ItemPrice) values('돼지 삼겹살' 	, '육류' , '25000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('돼지 목살' 		, '육류' , '27000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('한우 등심' 		, '육류' , '57000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('닭고기 7호' 	, '육류' , '19800');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('닭고기 8호' 	, '육류' , '25000');

INSERT INTO Items(ItemName , ItemType , ItemPrice) values('닭강정' 	, 'PB제품' , '9900');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('옛날통닭' 	, 'PB제품' , '19900');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('훈제삼겹살' , 'PB제품' , '29900');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('훈제오리'  , 'PB제품' , '25000');

INSERT INTO Items(ItemName , ItemType , ItemPrice) values('비비고냉동만두'  , '냉동식품' , '6000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('고향만두'  		, '냉동식품' , '7000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('풀무원물만두'  	, '냉동식품' , '9000');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('오뚜기OX만두'  	, '냉동식품' , '8000');

INSERT INTO Items(ItemName , ItemType , ItemPrice) values('오뚜기진라면매운맛'  	, '라면' , '990');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('오뚜기진라면순한맛'  	, '라면' , '890');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('신라면'  			, '라면' , '1100');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('신라면레드'  		, '라면' , '1250');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('팔도비민면'  		, '라면' , '980');

INSERT INTO Items(ItemName , ItemType , ItemPrice) values('코카콜라'  		, '주류' , '600');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('펩시'  		, '주류' , '1100');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('칠성사이다'  	, '주류' , '1200');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('솔의눈'  		, '주류' , '1500');
INSERT INTO Items(ItemName , ItemType , ItemPrice) values('밀키스'  		, '주류' , '1300');


```

## 시나리오 
총 5명의 사람이 금일 무작위로 물풀을 구매할 예정입니다 구매가 끝나면 해당내역을 정산 테이블로 이관하여서 구매 = 정산의 데이터가 올바른지 확인을 하는 시나리오를 만들것입니다 

## 사전작업 코드 (GitLab 주소 첨부)
<https://gitlab.com/kimdongy1000/spring-batch/-/tree/main/src/main/java/com/example/demo/batch/shpping?ref_type=heads>
해당 git 은 민감정보 (application.yml 을 제외한 모든 소스코드파일입니다)


## 실행시 
```
Users{userID=1, userName='KIM'}
Users{userID=2, userName='LEE'}
Users{userID=3, userName='TIME'}
Users{userID=4, userName='HEO'}
Users{userID=5, userName='BEAN'}
Items{itemId=1, itemName='수박', itemType='과일', itemPrice=1000}
Items{itemId=2, itemName='오렌지', itemType='과일', itemPrice=1500}
Items{itemId=3, itemName='청포도', itemType='과일', itemPrice=1700}
Items{itemId=4, itemName='포도', itemType='과일', itemPrice=2000}
Items{itemId=5, itemName='딸기', itemType='과일', itemPrice=2300}
Items{itemId=6, itemName='java 100일 문법', itemType='책', itemPrice=5000}
Items{itemId=7, itemName='oracle 100일 문법', itemType='책', itemPrice=6000}
Items{itemId=8, itemName='spring 이주일만에  끝내기', itemType='책', itemPrice=13000}
Items{itemId=9, itemName='보안이란 무엇인가', itemType='책', itemPrice=20000}
Items{itemId=10, itemName='알고리즘 입문', itemType='책', itemPrice=50000}
Items{itemId=11, itemName='돼지 삼겹살', itemType='육류', itemPrice=25000}
Items{itemId=12, itemName='돼지 목살', itemType='육류', itemPrice=27000}
Items{itemId=13, itemName='한우 등심', itemType='육류', itemPrice=57000}
Items{itemId=14, itemName='닭고기 7호', itemType='육류', itemPrice=19800}
Items{itemId=15, itemName='닭고기 8호', itemType='육류', itemPrice=25000}
Items{itemId=16, itemName='닭강정', itemType='PB제품', itemPrice=9900}
Items{itemId=17, itemName='옛날통닭', itemType='PB제품', itemPrice=19900}
Items{itemId=18, itemName='훈제삼겹살', itemType='PB제품', itemPrice=29900}
Items{itemId=19, itemName='훈제오리', itemType='PB제품', itemPrice=25000}
Items{itemId=20, itemName='비비고냉동만두', itemType='냉동식품', itemPrice=6000}
Items{itemId=21, itemName='고향만두', itemType='냉동식품', itemPrice=7000}
Items{itemId=22, itemName='풀무원물만두', itemType='냉동식품', itemPrice=9000}
Items{itemId=23, itemName='오뚜기OX만두', itemType='냉동식품', itemPrice=8000}
Items{itemId=24, itemName='오뚜기진라면매운맛', itemType='라면', itemPrice=990}
Items{itemId=25, itemName='오뚜기진라면순한맛', itemType='라면', itemPrice=890}
Items{itemId=26, itemName='신라면', itemType='라면', itemPrice=1100}
Items{itemId=27, itemName='신라면레드', itemType='라면', itemPrice=1250}
Items{itemId=28, itemName='팔도비민면', itemType='라면', itemPrice=980}
Items{itemId=29, itemName='코카콜라', itemType='주류', itemPrice=600}
Items{itemId=30, itemName='펩시', itemType='주류', itemPrice=1100}
Items{itemId=31, itemName='칠성사이다', itemType='주류', itemPrice=1200}
Items{itemId=32, itemName='솔의눈', itemType='주류', itemPrice=1500}
Items{itemId=33, itemName='밀키스', itemType='주류', itemPrice=1300}

```

이렇게 저장이 출력이 되면 올바르게 된것입니다 

## 무작위 쇼핑진행 
이제 등록된 사람들이 무작위로 쇼핑을 즐길것입니다 이때 우리는 이 데이터들을 Purchases 넣을것인데

```
@Configuration
@RequiredArgsConstructor
public class ShoppingBatchJob {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private PurchasesJpaRepository purchasesRepository;

    @Autowired
    private final List<Users> users;

    @Autowired
    private final List<Items> items;



    @Bean
    public Job shoppingJob(Step shoppingStep1){

        return jobBuilderFactory.get("shoppingJob")
                .incrementer(new RunIdIncrementer())
                .start(shoppingStep1)
                .build();
    }

    @Bean
    @JobScope
    public Step shoppingStep1(){

        return stepBuilderFactory.get("shoppingStep1")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {

                        Random userRnd = new Random();
                        Random itemRnd = new Random();


                        List<Purchases> purchasesList = new ArrayList<>();
                        String todayFormat = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));


                        for (int i = 0; i < 10000; i++) {

                            int user_rndNumber = userRnd.nextInt(4);
                            int item_rndNumber = itemRnd.nextInt(32);

                            Users rndUser = users.get(user_rndNumber);
                            Items rndItems = items.get(item_rndNumber);

                            purchasesList.add(new Purchases(todayFormat +"_"+ UUID.randomUUID().toString()  , rndUser.getUserID() , rndItems.getItemId() ,  new Date()));

                        }

                       purchasesRepository.saveAll(purchasesList);


                        return RepeatStatus.FINISHED;
                    }


                }).build();

    }
}

```
shoppingStep1 을 잘보시면 for 문을 통해서 무작위 경우 1만번을 실행시켜서 최종 1만번의 쇼핑을 시킵니다 그리고 그 결과를 list 에 담아두고 그 리스트를 save 하면됩니다 그러면 
DB 데이터를 보게 되면 

![ERD](/assets/img/post_img/batch_2023-04-22_19_00_2.png)

이렇게 무작위 데이터 1만개가 들어와 있는모습을 알 수 있습니다 여기 까지가 사전작업 이제 본 작업 실행하겠습니다 본작업은 다음장에서 진행을 하겠습니다