---
title: Spring Batch8 - Batch ItemReader , Itemprocessor , ItemWriter
author: kimdongy1000
date: 2023-04-22 20:00
categories: [Back-end, Spring - Batch]
tags: [ Spring , Bactch ]
math: true
mermaid: true
---

## 지난시간 리뷰
지난시간에는 batch에서 사용할 수 있는 기초 데이터를 쇼핑몰로 진행을 했습니다 금일부터 기초 데이터를 가지고 위에 3가지에 대해서 알아보도록 하겠습니다 

## 시나리오 점검 
구매테이블 Purchases 의 데이터를 가공해서 정산 테이블로 Calculates 로 옮기는 작업을 하게 됩니다 

## ItemReader
ItemReader는 Spring Batch에서 배치 작업에서 데이터를 읽어오는 역할을 담당하는 인터페이스입니다. 주로 파일, 데이터베이스, 메시지 큐 등 다양한 소스에서 데이터를 읽어와서 배치 작업에 전달하는 데 사용됩니다.

우리는 이중에서 데이터베이스에서 읽어오는 방식으로 진행을 하겠습니다 

## ItemProcessor
는 Spring Batch에서 사용되는 인터페이스로, 배치 작업의 각 항목을 가공하거나 변환하는 역할을 수행합니다. 보통 읽어온 데이터를 가공하여 새로운 형태로 변환하거나, 데이터의 일부를 걸러내거나 필터링하는 등의 작업을 수행합니다. 주로 ItemReader로부터 읽어온 데이터를 가공하고, 그 결과를 ItemWriter로 전달하기 전에 중간 단계에서 사용됩니다.

## ItemWriter
Spring Batch에서 사용되는 인터페이스로, 배치 작업에서 처리된 데이터를 저장하거나 외부 시스템에 전달하는 역할을 수행합니다. 일반적으로 ItemProcessor를 거친 데이터를 최종적으로 저장하는 데 사용됩니다.

## Batch 소스 코드
<https://gitlab.com/kimdongy1000/spring-batch/-/tree/main/src/main/java/com/example/demo/batch/shpping?ref_type=heads>

전체 소스코드는 이곳을 참고해주시고 이곳에는 main 로직만 설명하겠습니다 

```
@Configuration
@AllArgsConstructor
public class ShoppingBatchJob2 {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private PurchasesJpaRepository purchasesRepository;

    @Autowired
    private ItemsJpaRepository itemsJpaRepository;

    @Autowired
    private UserJpaRepository userJpaRepository;

    @Autowired
    private CalculatesJpaRepository calculatesJpaRepository;


    @Bean
    public Job shoppingJob2(Step shoppingStep2 ){

        return jobBuilderFactory.get("shoppingJob2")
                .incrementer(new RunIdIncrementer())
                .start(shoppingStep2)
                .build();
    }

    @Bean
    @JobScope
    public Step shoppingStep2(ItemReader<Purchases> purchasesReader , ItemProcessor<Purchases , Calculates> converterData , ItemWriter<Calculates> calculatesWriter){

        return stepBuilderFactory.get("shoppingStep2")
                .<Purchases , Calculates >chunk(100)
                .reader(purchasesReader)
                .processor(converterData)
                .writer(calculatesWriter)
                .build();

    }

    @Bean
    @StepScope
    public ItemReader<Purchases> purchasesReader(){

        return new RepositoryItemReaderBuilder<Purchases>()
                .name("purchasesReader")
                .repository(purchasesRepository)
                .methodName("findAll")
                .pageSize(100)
                .sorts(Collections.singletonMap("purchaseId" , Sort.Direction.ASC))
                .build();

    }

    @Bean
    @StepScope
    public ItemProcessor<Purchases , Calculates> converterData(){

        final String todayFormat = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));

        return purchases -> {

            Items items = itemsJpaRepository.findById(purchases.getItemId()).get();
            Users users = userJpaRepository.findById(purchases.getUserId()).get();



            return new Calculates(todayFormat + "-" + users.getUserID() + "-" + items.getItemName() + "-" + UUID.randomUUID()
                                    ,users.getUserID()
                                    ,items.getItemId()
                                    ,purchases.getItemId()
                                    ,users.getUserName()
                                    ,items.getItemName()
                                    ,items.getItemType()
                                    ,items.getItemPrice()
                                    ,purchases.getPurchaseIdDts()
            );
        };

    }

    @Bean
    @StepScope
    public ItemWriter<Calculates> calculatesWriter(){

        return new RepositoryItemWriterBuilder<Calculates>()
                .repository(calculatesJpaRepository)
                .methodName("save")
                .build();
    }
}


```

## step 
```

@Bean
@JobScope
public Step shoppingStep2(ItemReader<Purchases> purchasesReader , ItemProcessor<Purchases , Calculates> converterData , ItemWriter<Calculates> calculatesWriter){

    return stepBuilderFactory.get("shoppingStep2")
            .<Purchases , Calculates >chunk(100)
            .reader(purchasesReader)
            .processor(converterData)
            .writer(calculatesWriter)
            .build();

}

```
이제 step 는 tasklet 이 아닌 총 3개의 프로세가 일정하게 동작하도록 만들었습니다 위에서 본것처럼 ItemReader -> ItemProcessor -> ItemWriter 순으로 진행이 될것입니다 
중간에 `.<Purchases , Calculates >chunk(100)` 가 있는데 이때 chunk 는 한번에 읽을량과 쓸 량을 선언할 수 있습니다 그리고 제네릭 표현식은 `<읽어올 데이터 , 쓸 데이터>`
이렇게 정의를 해주게 됩니다 

## ItemReader
```

@Bean
@StepScope
public ItemReader<Purchases> purchasesReader(){

    return new RepositoryItemReaderBuilder<Purchases>()
            .name("purchasesReader")
            .repository(purchasesRepository)
            .methodName("findAll")
            .pageSize(100)
            .sorts(Collections.singletonMap("purchaseId" , Sort.Direction.ASC))
            .build();

}

```
ItemReader 같은 경우에는 어디에서 데이터를 읽어오냐입니다 우리는 Purchases 타입으로 ItemReader 을 만들것이고 이때 사용하는 소스코드는 jpa 를 이용한 
ItemReader 의 구현체를 사용할 것입니다 (RepositoryItemReaderBuilder) 메서드 명만보더라도 이 소스가 어떤 일을 하는지 알 수 있습니다 

## ItemProcessor

```

@Bean
@StepScope
public ItemProcessor<Purchases , Calculates> converterData(){

    final String todayFormat = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));

    return purchases -> {

        Items items = itemsJpaRepository.findById(purchases.getItemId()).get();
        Users users = userJpaRepository.findById(purchases.getUserId()).get();



        return new Calculates(todayFormat + "-" + users.getUserID() + "-" + items.getItemName() + "-" + UUID.randomUUID()
                                ,users.getUserID()
                                ,items.getItemId()
                                ,purchases.getItemId()
                                ,users.getUserName()
                                ,items.getItemName()
                                ,items.getItemType()
                                ,items.getItemPrice()
                                ,purchases.getPurchaseIdDts()
        );
    };

}

```
ItemProcessor 는 데이터를 가공하는 과정입니다 우리가 읽어온 Purchases -> Calculates 옮겨야 하는 작업을 진행하는데 이때 사용하는 방법이 바로 ItemProcessor 를 활용해서 데이터를 가공하고 있는 과정입니다 가공과정은 단순합니다 각각의 값을 읽어서 DB 에 넣을 수 있게 도메인을 만들어서 return 하는 과정입니다 

## ItemWriter
```

@Bean
@StepScope
public ItemWriter<Calculates> calculatesWriter(){

    return new RepositoryItemWriterBuilder<Calculates>()
            .repository(calculatesJpaRepository)
            .methodName("save")
            .build();
}

```
ItemWriter 는 이제 ItemProcessor 에서 가공이 완료된 데이터를 이제 다시 쓰기 위해서 사용할 수 있습니다 이때 쓰기는 단순 DB 쓰기를 포함한 엑셀 데이터 다운로드 같은 것도 포함입니다 그런것들은 다음장에서 해보도록 하겠습니다 마찬가지로 ItemWriter 구현체 (RepositoryItemWriterBuilder) 를 사용하고 있습니다 이렇게 진행을 하게 되면 

```

...

Hibernate: insert into Calculates (ItemId, ItemName, ItemPrice, ItemType, PurchaseId, PurchaseIdDts, UserId, UserName, CalculateId) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into Calculates (ItemId, ItemName, ItemPrice, ItemType, PurchaseId, PurchaseIdDts, UserId, UserName, CalculateId) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into Calculates (ItemId, ItemName, ItemPrice, ItemType, PurchaseId, PurchaseIdDts, UserId, UserName, CalculateId) values (?, ?, ?, ?, ?, ?, ?, ?, ?)

...

```
가공된 데이터들이 이렇게 jpa 를 통해서 insert 가 됩니다 그럼 DB 를 살펴보면

![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/2ef7c805-92da-4f05-9d84-cb8586cbee2d)

이렇게 데이터가 이쁘게 들어간것을 확인할 수 있습니다 우리는 이렇게 오늘 단순 선형작업인 tasklet 에서 ItemReader , ItemProcessor , ItemWriter 에 대해서 알아보았습니다 다음장에서는 좀더 다양한 ItemWriter 에 대해서 알아보도록 하겠습니다 