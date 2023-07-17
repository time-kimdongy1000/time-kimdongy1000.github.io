---
title: Spring Batch3
author: kimdongy1000
date: 2023-04-24 11:43
categories: [Back-end, Spring - MVC]
tags: [ MVC , Bactch ]
math: true
mermaid: true
---

## Batch
우리는 지난번 batch 가 무엇인지 알아보고 간단한 static list 에 데이터를 담아서 사용하는 방법을 그리고 batch2 를 통해서 DB 에 데이터를 읽어서 다른 테이블로 옮기는 작업을 그리고 오늘 할것은 좀더 많은 데이터로 이 작업을 진행을 하겠습니다.

먼제 데이터를 만들겠습니다 

```

@Service
public class EmpService {
	
	private static final String emp_prefix = "emp";
	
	
	@Autowired
	private EmpRepository empRepository;
	
	
	public void insertEmpUser() {
		
		List<EmpDto> dtos = new ArrayList<EmpDto>();
		
		for(int i = 0; i < 10000; i++) {
			EmpDto dto = new EmpDto(emp_prefix + Integer.toString(i) , UUID.randomUUID().toString().substring(0, 15));
			dtos.add(dto);
		}
		
		
		empRepository.lotInsert(dtos);
				
	}

}

```

```

public class EmpDto {
	
	private String name;
	
	private String emp_no;
	
	public EmpDto() {
	}

	public EmpDto(String name, String emp_no) {
		this.name = name;
		this.emp_no = emp_no;
	}
}


```

우리가 만든 데이터는 총 1만개로 스키마 모양이나 어떤 데이터를 넣을것인지 보일것이다 시나리오를 써보자 

외부의 공격으로 우리 사원의 정보가 노출이 되었습니다 이를 수습하기 위해 기존 DB 를 폐기하고 새로운 스키마에 정보를 담을것인데 이전 정보를 name 말고 emp_no 를 새롭게 만드는 시나리오 입니다 

자 그럼 batch 를 만들어봅시다 

사실 이전내용하고 동일합니다 이 동일한 내용을 DB 량만 변경해서 하는 이유는 얼마나 많은데이터를 효율적이고 쉽게 변경할 수 있는것인지 그리고 이전에 하지 못한 중간 데이터 변경까지 하는 작업이라서 따로 네이밍을 따서 진행을 합니다 


## 전체소스 
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

                Random random = new Random();
                long _long = random.nextLong();

                item.setEmp_no(Long.toString(_long));

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
                .<EmpDto, EmpDto>chunk(1000) // 청크 크기 설정
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

컨셉은 동일함으로 전체 소스로 보겠습니다 

```

@Bean
public ItemProcessor<EmpDto, EmpDto> processor() {
    return new ItemProcessor<EmpDto, EmpDto>() {
        @Override
        public EmpDto process(EmpDto item) {

            Random random = new Random();
            long _long = random.nextLong();

            item.setEmp_no(Long.toString(_long));

            return item;
        }
    };
}

```
바뀐 부분은 이 부분입니다 이 부분에서 현재 데이터를 가공해서 새로운곳에 넣고 있고 그 데이터를 EMP2 테이블에 밀어넣고 있습니다 이를 통해서 우리는 안전하게 새로운 스키마에 데이터를 집어 넣었습니다 

그리고 그 결과는 아래와 같습니다 

![1](https://github.com/SH-Yeon93/ImageStore/assets/58513678/2260108d-5006-4910-93ac-1d33ea0ecd7a)

![2](https://github.com/SH-Yeon93/ImageStore/assets/58513678/613f29fd-cc8b-4ce0-a1fb-8246696f14e7)

이렇게 바뀔 예정입니다 우리는 이렇게 대용량 데이터도 만져 보았습니다 
다음 포스터는 마지막으로 이 batch 를 통해서 통계를 만들어내는 작업을 끝으로 Batch 프로젝트는 마무리 짓도록 하겠습니다 










