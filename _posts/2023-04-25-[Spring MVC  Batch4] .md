---
title: Spring Batch4
author: kimdongy1000
date: 2023-04-25 11:43
categories: [Back-end, Spring - MVC]
tags: [ MVC , Bactch ]
math: true
mermaid: true
---

## Batch
우리는 Srping - Batch 를 통해서 정말 많은 것을 해보았다 물론 비교적 쉽고 간단한것들을 해서 이것들이 실무에서 어떻게 쓰이는지 감을 못잡겠지만 이 포스터의 기본부터 조금씩 따라오다 보면 
기본을 통해서 더욱더 나은 프로그램을 만들 수 있을것이다 결국 복잡함의 출발은 가장간단한 것을 구축하는것 부터이다 
오늘은 batch 의 마지막으로 통계를 만들어낼것이다 시나리오는 이렇게 된다 

기본틀은 로또이다 우리는 하나의 row 당 중복되지 않은 1번 부터 45번까지 번호를 6개 뽑아내서 저장을 해놓을것이다 이때 개수는 약10만개를 저장하고 
이 10만개를 추출해서 어떤 번호가 몇번 나왔는지에 대한 프로그램을 만들것이다 결국은 2개의 테이블이 존재한다 

하나의 테이블은 컬럼이 6개인 테이블은데 하나의 row 의 중복되지 않은 6개의 숫자가 10만 row 가 쌓일것이고 
두번쨰 테이블은 컬럼이 45개인 테이블인데 각각의 컬러마다 숫자가 쓰여져 있어 해당 숫자공이 몇번 나왔는지를 업데이트 칠것이다 

조금 까다로운 조건이긴 하지만 결론은 어떻게 데이터를 만들어내어서 어떻게 저장하는것에 달린것이다 그럼 보자 

## 로또 프로그램 작성 
좀 내용이 돌아가지만 먼저 로또 프로그램을 작성을 해보자 목적은 하나의 게임당 1번부터 45번까지 총 45개의 공을 뽑는것으로 동일한 확률보다는 단순 뽑기를 했들때 이미 뽑힌 숫자가 있으면 
새로 뽑는 방식으로 진행하겠습니다 

## ProtoType Mode 
먼저 샘플을 작성을 해보자 가볍게 100개의 게임을 한다고 가정을 했을때 이와 같이 쓰일것이다 

```

@Service
public class LotNumberService {

	private static final int static_number[] = {

			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29,
			30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45 };
	
	
	@Autowired
	private LotNumberRepository lotNumberRepository;

	public void insertLotNumber() {

		List<LotNumberDto> dtos = new ArrayList<LotNumberDto>();
		List<Integer> tem_inter = new ArrayList<Integer>();

		Random random = new Random();

		for (int i = 0; i < 100; i++) {
			
			LotNumberDto dto = new LotNumberDto();

			for (int j = 1; j <= 6; j++) {
				
				

				int rnd_int = random.nextInt(45);
				int select_number = static_number[rnd_int];

				if (tem_inter.contains(select_number)) {
					j = j - 1;
					continue;
					
				} else {

					tem_inter.add(select_number);

					switch (j) {
					
					case 1:
						dto.setFirstNmuber(select_number);
						break;

					case 2:
						dto.setSecondsNumber(select_number);
						break;

					case 3:
						dto.setThirdsNumber(select_number);
						break;

					case 4:
						dto.setForthNumber(select_number);
						break;

					case 5:
						dto.setFifthNumber(select_number);
						break;

					case 6:
						dto.setSixthNumber(select_number);
						break;


					}
					
				}

			}
			dtos.add(dto);
			tem_inter.clear();
			

		}
		
		lotNumberRepository.insertBatch(dtos);
	}

}

```

```

<insert id = "insertLotNumber" parameterType="java.util.List">
    INSERT ALL

    <foreach collection="list" item="number">
        INTO Lot_Number_project (firstNumber , secondsNumber , thirdsNumber , forthsNumber , fifthNumber , sixthNumber) VALUES
        (#{number.firstNmuber}, #{number.secondsNumber}, #{number.thirdsNumber} ,  #{number.forthNumber} ,  #{number.fifthNumber} ,  #{number.sixthNumber})
    </foreach>
    SELECT * FROM dual
</insert>

```

```
package com.cybb.main.repository;


@Repository
public class LotNumberRepository {
	
    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;



    public void insertBatch(List<LotNumberDto> queryParam) {

        SqlSessionFactory sqlSessionFactory = sqlSessionTemplate.getSqlSessionFactory();
        SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);

        try{
            sqlSessionTemplate.insert(this.getClass().getName() + ".insertLotNumber" , queryParam);
            session.flushStatements(); // 배치 작업 실행
            session.commit();
        }catch(Exception e){
            throw new RuntimeException(e);

        }finally {
            session.close();
            System.out.println("종료입니다");

        }

    }
}

```

대강 이렇게 보면 무슨 일을 하고 스키마가 어떻게 될것인지 알것이다 그렇게 게임을 만들면 다음처럼 데이터가 생길것이다 

![3](https://github.com/SH-Yeon93/ImageStore/assets/58513678/f1011572-21ba-4675-9ef2-256eaee923d2)

이렇게 한 게임당 중복되지 않은 숫자를 구할 수 있을것이다 

그럼 우리는 일단 이 100개 가디고 Batch 를 통해서 통계를 만들어보자 

그전에 이 count 의 개수를 세는 테이블을 만들어보자 좀 무식하긴 하지만 1부터 45까지 컬럼으로 만든것이다 

```

CREATE TABLE "KIMDONGY1000"."LOT_NUMBER_PROJECT_TEMP" 
   (	"T1" NUMBER, 
	"T2" NUMBER, 
	"T3" NUMBER, 
	"T4" NUMBER, 
	"T5" NUMBER, 
	"T6" NUMBER, 
	"T7" NUMBER, 
	"T8" NUMBER, 
	"T9" NUMBER, 
	"T10" NUMBER, 
	"T11" NUMBER, 
	"T12" NUMBER, 
	"T13" NUMBER, 
	"T14" NUMBER, 
	"T15" NUMBER, 
	"T16" NUMBER, 
	"T17" NUMBER, 
	"T18" NUMBER, 
	"T19" NUMBER, 
	"T20" NUMBER, 
	"T21" NUMBER, 
	"T22" NUMBER, 
	"T23" NUMBER, 
	"T24" NUMBER, 
	"T25" NUMBER, 
	"T26" NUMBER, 
	"T27" NUMBER, 
	"T28" NUMBER, 
	"T29" NUMBER, 
	"T30" NUMBER, 
	"T31" NUMBER, 
	"T32" NUMBER, 
	"T33" NUMBER, 
	"T34" NUMBER, 
	"T35" NUMBER, 
	"T36" NUMBER, 
	"T37" NUMBER, 
	"T38" NUMBER, 
	"T39" NUMBER, 
	"T40" NUMBER, 
	"T41" NUMBER, 
	"T42" NUMBER, 
	"T43" NUMBER, 
	"T44" NUMBER, 
	"T45" NUMBER
   ) 
  
```

정말 원초적이라서 딱히 할말은 없지만 그래도 이 방법이 최고인듯 하다 

그럼 우리의 시나리오는 이렇게 앞의 게임을 분석해서 1이 몇개 2가 몇개 3이 몇개 들어가 있는지 분석을 한 뒤에 저 테이블에 해당 개수만큼 insert 시켜주기만 하면된다 
결국은 번호가 몇개인지만 잘 추출하면되는것이다 그럼 먼저 게임을 분석을 해보자 

사실 이 글로 마지막을 할려고 했는데 몇가지 필요한 기능을 좀더 공부를 해야 겠다 그에 대한 내용은 다시 정리 예정이다 일단 한번보자 

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

    private final MainLotNumber2 mainLotNumber2 = new MainLotNumber2();

    @Bean
    public ItemReader<MainLotNumber> reader() {
        JdbcCursorItemReader<MainLotNumber> reader = new JdbcCursorItemReader<>();
        reader.setDataSource(dataSource);
        reader.setSql("SELECT FIRSTNUMBER , SECONDSNUMBER ,THIRDSNUMBER , FORTHSNUMBER , FIFTHNUMBER , SIXTHNUMBER FROM LOT_NUMBER_PROJECT");
        reader.setRowMapper(new BeanPropertyRowMapper<>(MainLotNumber.class));
        reader.setFetchSize(100);
        return reader;
    }


    @Bean
    public ItemProcessor<MainLotNumber, MainLotNumber2> processor() {
        return new ItemProcessor<MainLotNumber, MainLotNumber2>() {
            @Override
            public MainLotNumber2 process(MainLotNumber item) {

                int firstNumber = item.getFIRSTNUMBER();
                int secondsNumber = item.getSECONDSNUMBER();
                int thirdNumber = item.getTHIRDSNUMBER();
                int forthNumber = item.getFORTHSNUMBER();
                int fifthNumber = item.getFIFTHNUMBER();
                int sixthNumber = item.getSIXTHNUMBER();

                List<Integer> integers_list = new ArrayList<>();

                integers_list.add(firstNumber);
                integers_list.add(secondsNumber);
                integers_list.add(thirdNumber);
                integers_list.add(forthNumber);
                integers_list.add(fifthNumber);
                integers_list.add(sixthNumber);


                for(int i = 0; i < integers_list.size(); i++){

                    if(integers_list.get(i) == 1) mainLotNumber2.setT1(mainLotNumber2.getT1() + 1);
                    if(integers_list.get(i) == 2) mainLotNumber2.setT2(mainLotNumber2.getT2() + 1);
                    if(integers_list.get(i) == 3) mainLotNumber2.setT3(mainLotNumber2.getT3() + 1);
                    if(integers_list.get(i) == 4) mainLotNumber2.setT4(mainLotNumber2.getT4() + 1);
                    if(integers_list.get(i) == 5) mainLotNumber2.setT5(mainLotNumber2.getT5() + 1);
                    if(integers_list.get(i) == 6) mainLotNumber2.setT6(mainLotNumber2.getT6() + 1);
                    if(integers_list.get(i) == 7) mainLotNumber2.setT7(mainLotNumber2.getT7() + 1);
                    if(integers_list.get(i) == 8) mainLotNumber2.setT8(mainLotNumber2.getT8() + 1);
                    if(integers_list.get(i) == 9) mainLotNumber2.setT9(mainLotNumber2.getT9() + 1);
                    if(integers_list.get(i) == 10) mainLotNumber2.setT10(mainLotNumber2.getT10() + 1);
                    if(integers_list.get(i) == 11) mainLotNumber2.setT11(mainLotNumber2.getT11() + 1);
                    if(integers_list.get(i) == 12) mainLotNumber2.setT12(mainLotNumber2.getT12() + 1);
                    if(integers_list.get(i) == 13) mainLotNumber2.setT13(mainLotNumber2.getT13() + 1);
                    if(integers_list.get(i) == 14) mainLotNumber2.setT14(mainLotNumber2.getT14() + 1);
                    if(integers_list.get(i) == 15) mainLotNumber2.setT15(mainLotNumber2.getT15() + 1);
                    if(integers_list.get(i) == 16) mainLotNumber2.setT16(mainLotNumber2.getT16() + 1);
                    if(integers_list.get(i) == 17) mainLotNumber2.setT17(mainLotNumber2.getT17() + 1);
                    if(integers_list.get(i) == 18) mainLotNumber2.setT18(mainLotNumber2.getT18() + 1);
                    if(integers_list.get(i) == 19) mainLotNumber2.setT19(mainLotNumber2.getT19() + 1);
                    if(integers_list.get(i) == 20) mainLotNumber2.setT20(mainLotNumber2.getT20() + 1);
                    if(integers_list.get(i) == 21) mainLotNumber2.setT21(mainLotNumber2.getT21() + 1);
                    if(integers_list.get(i) == 22) mainLotNumber2.setT22(mainLotNumber2.getT22() + 1);
                    if(integers_list.get(i) == 23) mainLotNumber2.setT23(mainLotNumber2.getT23() + 1);
                    if(integers_list.get(i) == 24) mainLotNumber2.setT24(mainLotNumber2.getT24() + 1);
                    if(integers_list.get(i) == 25) mainLotNumber2.setT25(mainLotNumber2.getT25() + 1);
                    if(integers_list.get(i) == 26) mainLotNumber2.setT26(mainLotNumber2.getT26() + 1);
                    if(integers_list.get(i) == 27) mainLotNumber2.setT27(mainLotNumber2.getT27() + 1);
                    if(integers_list.get(i) == 28) mainLotNumber2.setT28(mainLotNumber2.getT28() + 1);
                    if(integers_list.get(i) == 29) mainLotNumber2.setT29(mainLotNumber2.getT29() + 1);
                    if(integers_list.get(i) == 30) mainLotNumber2.setT30(mainLotNumber2.getT30() + 1);
                    if(integers_list.get(i) == 31) mainLotNumber2.setT31(mainLotNumber2.getT31() + 1);
                    if(integers_list.get(i) == 32) mainLotNumber2.setT32(mainLotNumber2.getT32() + 1);
                    if(integers_list.get(i) == 33) mainLotNumber2.setT33(mainLotNumber2.getT33() + 1);
                    if(integers_list.get(i) == 34) mainLotNumber2.setT34(mainLotNumber2.getT34() + 1);
                    if(integers_list.get(i) == 35) mainLotNumber2.setT35(mainLotNumber2.getT35() + 1);
                    if(integers_list.get(i) == 36) mainLotNumber2.setT36(mainLotNumber2.getT36() + 1);
                    if(integers_list.get(i) == 37) mainLotNumber2.setT37(mainLotNumber2.getT37() + 1);
                    if(integers_list.get(i) == 38) mainLotNumber2.setT38(mainLotNumber2.getT38() + 1);
                    if(integers_list.get(i) == 39) mainLotNumber2.setT39(mainLotNumber2.getT39() + 1);
                    if(integers_list.get(i) == 40) mainLotNumber2.setT40(mainLotNumber2.getT40() + 1);
                    if(integers_list.get(i) == 41) mainLotNumber2.setT41(mainLotNumber2.getT41() + 1);
                    if(integers_list.get(i) == 42) mainLotNumber2.setT42(mainLotNumber2.getT42() + 1);
                    if(integers_list.get(i) == 43) mainLotNumber2.setT43(mainLotNumber2.getT43() + 1);
                    if(integers_list.get(i) == 44) mainLotNumber2.setT44(mainLotNumber2.getT44() + 1);
                    if(integers_list.get(i) == 45) mainLotNumber2.setT45(mainLotNumber2.getT45() + 1);



                }



                return mainLotNumber2;
            }
        };
    }

    @Bean
    public ItemWriter<MainLotNumber2> writer() {
        JdbcBatchItemWriter<MainLotNumber2> writer = new JdbcBatchItemWriter<>();
        writer.setDataSource(dataSource);
        writer.setSql("INSERT INTO LOT_NUMBER_PROJECT_TEMP (T1,T2,T3,T4,T5,T6,T7,T8,T9,T10,T11,T12,T13,T14,T15,T16,T17,T18,T19,T20,T21,T22,T23,T24,T25,T26,T27,T28,T29,T30,T31,T32,T33,T34,T35,T36,T37,T38,T39,T40,T41,T42,T43,T44,T45 ,add_date) VALUES (:T1,:T2,:T3,:T4,:T5,:T6,:T7,:T8,:T9,:T10,:T11,:T12,:T13,:T14,:T15,:T16,:T17,:T18,:T19,:T20,:T21,:T22,:T23,:T24,:T25,:T26,:T27,:T28,:T29,:T30,:T31,:T32,:T33,:T34,:T35,:T36,:T37,:T38,:T39,:T40,:T41,:T42,:T43,:T44,:T45 , TO_CHAR(sysdate , 'YYYYMMDD'))");
        writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
        return writer;
    }

    @Bean
    public Step myStep(ItemReader<MainLotNumber> reader, ItemProcessor<MainLotNumber, MainLotNumber2> processor, ItemWriter<MainLotNumber2> writer) {
        return stepBuilderFactory.get("myStep")
                .<MainLotNumber, MainLotNumber2>chunk(1) // 청크 크기 설정
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

예를 들면 이렇게 될것이다 딱히 앞에서 했던 예제 emp 에서 벗어나지 않으며 단순히 Proccess 에서 어떻게 하나의 원테이블로 넣냐에 달린것이다 다만 현재 이 코드는 문제가 있다 
돌려본 결과 

```

@Bean
public ItemReader<MainLotNumber> reader() {
	JdbcCursorItemReader<MainLotNumber> reader = new JdbcCursorItemReader<>();
	reader.setDataSource(dataSource);
	reader.setSql("SELECT FIRSTNUMBER , SECONDSNUMBER ,THIRDSNUMBER , FORTHSNUMBER , FIFTHNUMBER , SIXTHNUMBER FROM LOT_NUMBER_PROJECT");
	reader.setRowMapper(new BeanPropertyRowMapper<>(MainLotNumber.class));
	reader.setFetchSize(100);
	return reader;
}

```

이 ItemReader 는 한번에 하나의 데이터만 가지고 오게 된다 그렇기에 이 결과DB 를 살펴보면 다음과 같다 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/f8b74470-6a0b-4d76-a471-f61ab69f5c4d) 이렇게 될것이다 다만 이러한 것은 쿼리로 얼마든지 커버를 칠 수 있다 우리는 
맨뒤에 날짜 컬럼을 넣음으로서 하루에 한번 돌렸을때 결과만 보면되는것이다 그렇기에 

```

SELECT MAX(T1) , 
MAX(T2) ,
MAX(T3) ,
MAX(T4) ,
MAX(T5) ,
MAX(T6) ,
MAX(T7) ,
MAX(T8) ,
MAX(T9) ,
MAX(T10),
MAX(T11),
MAX(T12),
MAX(T13),
MAX(T14),
MAX(T15),
MAX(T16),
MAX(T17),
MAX(T18),
MAX(T19),
MAX(T20),
MAX(T21),
MAX(T22),
MAX(T23),
MAX(T24),
MAX(T25),
MAX(T26),
MAX(T27),
MAX(T28),
MAX(T29),
MAX(T30),
MAX(T31),
MAX(T32),
MAX(T33),
MAX(T34),
MAX(T35),
MAX(T36),
MAX(T37),
MAX(T38),
MAX(T39),
MAX(T40),
MAX(T41),
MAX(T42),
MAX(T43),
MAX(T44),
MAX(T45)

FROM LOT_NUMBER_PROJECT_TEMP	
GROUP BY ADD_DATE

```
쿼리는 이렇게 무식한 형태가 나올것이고 결과는 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/17e7dd1f-68a6-4ba4-8a3f-142c5e01ba25)

각각의 번호가 몇번 나오는지 만들어주는 통계가 완료된것이다 그럼 이는 100개만 했는데 우리는 그럼 메인 테이블에 백만개를 넣어보자 
얼마나 걸리는지 

자 그럼 데이터를 넣고 가공하는동안 무엇이 문제인지 한번 써보겠다 

일단 기본적으로 reader.setRowMapper(new BeanPropertyRowMapper<>(MainLotNumber.class)); 는 한번에 하나의 데이터만 가지고 오기 때문에 결국은 쿼리에서 무엇인가를 해야 한다 
게다가 데이터가 많으면 많을 수록 하나씩 뽑아오는건 역시나 다른 방법 또는 좀더 최적화를 해서 진행을 해야 한다 

우리가 백만개를 넣을때 걸린시간은 20분정도 걸렸고 이를 가공하는데 걸리는 시간은 좀걸린다 다음이 이를 고도화 할떄는 이 내용도 첨가를 해서 진행을 해야 겠다 








