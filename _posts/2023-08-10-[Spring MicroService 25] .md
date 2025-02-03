---
title: Spring MicroService 25 Spring MicroService 분산 캐싱
author: kimdongy1000
date: 2023-08-10 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService ,  MessageQ]
math: true
mermaid: true
---

우리는 지난시간에 비동기 메세지를 처리하면서 카프카와 주키퍼를 활용해서 메세지 생산자 , 메세지 소비자를 만들어 empClient 에서 empClient 의 생성 , 삭제 , 읽기 , 수정이 일어났을때 메세지 발행 , 그리고 그 메세지를 savingMoney 에서 소비하는 모습을 보였습이다 지난시간에는 메세지를 직접 발행 소비만 하는 과정을 보였지 이 과정이 왜 결합을 약하게 만들 수 있는지는 이번시간에 다루어볼려고 합니다 

## 아키텍쳐

![1](https://github.com/user-attachments/assets/7143d01a-e139-4ed9-9ed8-f5bf598822f6)

우리는 이제 EmpClient 의 데이터베이스를 캐싱할 DB 를 만들어둘것입니다 그리고 SavingMoney 는 EmpClient 에서 직접 데이터를 가져오지 않고 캐싱된 DB EmpClient 의 DB 의 데이터를 가져와서 로직을 처리합니다 그리고 EmpClient 에서 emp 데이터 변경시 메세지를 발행 , 그리고 SavingMoney 에서 해당 메세지를 통해서 캐싱 데이터를 무효화 하고 업데이트 하는 과정을 만들것입니다 이로 인해서 이전에는 EmpClient 와 강한결합이었던 SavingMoney 모듈이 이제 느슨하 결합으로 바뀌어 EmpClient 의 서버가 문제가 발생해도 영향이 덜가는 방향으로 만들어지게 됩니다 

## Redis 
우리가 분산캐싱DB 를 사용할때 사용할 데이터베이스입니다 특징은 key-value 형태로 데이터가 저장되며 매우 빠른 읽기 쓰기 성능을 제공하는것이 장점인 경량 DB 의 일종입니다 그 외의 redis 는 분산 클러스트링 의 세션관리 등등 다양한 방면에서 사용되고 있지만 이번시간에는 제일 대표적으로 활용되고 있는 캐싱에 대해서 쓸려고 합니다 

## 전체소스 
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project-messageQ-redis?ref_type=heads>

## Docker Redis 추가 
```

 redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data      
    command: ["redis-server", "--requirepass", "1234567890", "--appendonly", "yes"]      
    container_name: redis    

```
우리는 위에서 말했다 싶히 캐싱 DB 로 redis 를 사용할것이기 때문에 이미지를 docker 에 만들어줍니다 

## config-server savingMoney-dev.yml
```
spring:
  redis:
    datasource:
      host: redis
      port: 6379
      password: 1234567890

```

config 서버에 redis 를 선언을 해주겠습니다 

## miniProjectSavingMoney pom.xml
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/com.google.code.gson/gson -->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
</dependency>

```
의존성을 추가해줍니다 이때 gson 은 key-value 형태에서는 String 이 유리하기 때문에 이들을 변환해주는 역활을 하것입니다 

## miniProjectSavingMoney RedisConfig

```
@Configuration
public class RedisConfig {

    @Value("${spring.redis.datasource.host}")
    private String host;

    @Value("${spring.redis.datasource.port}")
    private int port;

    @Value("${spring.redis.datasource.password}")
    private String password;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {

        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(host);
        redisStandaloneConfiguration.setPort(port);
        redisStandaloneConfiguration.setPassword(password);

        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration);


        return lettuceConnectionFactory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(){

        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());

        return redisTemplate;
    }
}


```
먼저 redis 와 통신할 Bean 을 생성하겠습니다

## miniProjectSavingMoney RedisRepository

```
@Repository
public class RedisRepository {


    @Autowired
    private RedisTemplate<String , Object> redisTemplate;


    public void insert_redis_data(String key , Object value , long timeOut , TimeUnit timeUnit){
        redisTemplate.opsForValue().set(key , value , timeOut , timeUnit);
    }

    public Object select_redis_data(String key){
        return redisTemplate.opsForValue().get(key);
    }

    public boolean delete_redis_data(String key){
        return redisTemplate.delete(key);
    }

    public boolean expire_redis_data(String key , long timeOut , TimeUnit timeUnit){
        return redisTemplate.expire(key , timeOut , timeUnit);
    }

    public Set<String> select_redis_all_keys(){
        return redisTemplate.keys("*");
    }

}



```
redisTemplate 이 사용할 curd 메서드를 만들어주겠습니다 

## miniProjectSavingMoney EmpClientRedisTemplate
```
@Component
public class EmpClientRedisTemplate {

    private static final Logger log = LoggerFactory.getLogger(EmpClientRedisTemplate.class);

    @Autowired
    private RedisRepository redisRepository;

    @Autowired
    private SearchEurekaService searchEurekaService;

    @Autowired
    private Gson gson;

    public EmpDao checkEmpRedisData(String emp_id)
    {
        try{

            log.info("===============================================");
            log.info("checkEmpRedisData : {}" , emp_id);
            log.info("===============================================");

            String str_empDao = (String)redisRepository.select_redis_data(emp_id);
            return gson.fromJson(str_empDao, EmpDao.class);
        }catch(Exception e){
            log.error(e.getMessage());
            return null;
        }
    }

    public void saveEmpRedisData(EmpDao empDao)
    {
        try{

            log.info("===============================================");
            log.info("saveEmpRedisData : ");
            log.info("===============================================");

            String str_emp = gson.toJson(empDao);

            redisRepository.insert_redis_data(empDao.getEmp_id() , str_emp , 3600 , TimeUnit.SECONDS);
        }catch (Exception e){
            log.error(e.getMessage());

        }

    }

    public EmpDao getEmpRedisData(String emp_id)
    {
        try{

            log.info("===============================================");
            log.info("getEmpRedisData : {}" , emp_id);
            log.info("===============================================");

            EmpDao empDao = checkEmpRedisData(emp_id);

            if(empDao != null){
                return empDao;

            }else{

                String serviceUrl = searchEurekaService.getInstancesUri("miniProject-EmpClient");
                RestTemplate restTemplate = new RestTemplate();
                String requestUrl = String.format("%s/readEmp/%s", serviceUrl , emp_id);

                ResponseEntity<EmpDao> restExchange = restTemplate.exchange(
                        requestUrl,
                        HttpMethod.GET,
                        null ,
                        EmpDao.class
                );

                empDao = restExchange.getBody();
                saveEmpRedisData(empDao);
                return empDao;
            }

        }catch(Exception e){
            log.error(e.getMessage());
            return null;
        }

    }

    public void deleteEmpRedisData(String emp_id)
    {
        try{

            log.info("===============================================");
            log.info("deleteEmpRedisData : {}" , emp_id);
            log.info("===============================================");

            EmpDao empDao = checkEmpRedisData(emp_id);
            if(empDao != null){
                redisRepository.delete_redis_data(emp_id);
            }

        }catch(Exception e){
            log.error(e.getMessage());
        }
    }

    public void updateEmpRedisData(String emp_id , EmpDao empDao)
    {
        try{

            deleteEmpRedisData(emp_id);
            saveEmpRedisData(empDao);


        }catch(Exception e){
            log.error(e.getMessage());
        }

    }

    public Set<String> getAllEmp_id(){
        return redisRepository.select_redis_all_keys();
    }
}


```

이 부분이 핵심입니다 redisRepository 을 통해서 redis 에서 데이터를 체크하는 `checkEmpRedisData` 그리고 저장을 하는 `saveEmpRedisData` 그리고 업데이트 하는 `updateEmpRedisData` 
삭제하는 `deleteEmpRedisData` 그리고 데이터를 가져오는 `getEmpRedisData` 들을 작성을 합니다 이 클래스를 통해서 empRedis 데이터를 관리할것입니다 

## miniProjectSavingMoney MiniProjectSavingMoneyApplication

```
@SpringBootApplication
public class MiniProjectSavingMoneyApplication {

	@Autowired
	private EmpClientRedisTemplate empClientRedisTemplate;

	private final static Logger log = LoggerFactory.getLogger(MiniProjectSavingMoneyApplication.class);

	public static void main(String[] args) {
		SpringApplication.run(MiniProjectSavingMoneyApplication.class, args);
	}

	@StreamListener("inputChannel")
	public void loggerSink(EmpChangeModel empChangeModel)
	{
		log.info("Received Message To EmpClient : {} , {}"  , empChangeModel.getStatus() , empChangeModel.getEmd_id());

		switch (empChangeModel.getStatus()) {
			case "CREATE":
				empClientRedisTemplate.saveEmpRedisData(empChangeModel.getEmpDao());
				break;
			case "READ":
				break;

			case "UPDATE":
				empClientRedisTemplate.updateEmpRedisData(empChangeModel.getEmd_id() , empChangeModel.getEmpDao());
				break;

			case "DELETE":
				empClientRedisTemplate.deleteEmpRedisData(empChangeModel.getEmd_id());
				break;
		}
	}
}


```
지난시간에는 단순 로그만 찍었다면 이제는  `empChangeModel.getStatus()` 를 통해서 캐싱작업을 진행을 하게 됩니다 

## miniProjectSavingMoney SavingMoneyController

```

@PostMapping("/createMoney")
public ResponseEntity<?> createMoney()
{
    try{


        Set<String> all_emp_id_set = empClientRedisTemplate.getAllEmp_id();
        Random rnd = new Random();
        int int_rnd = rnd.nextInt(all_emp_id_set.size());
        List<String> all_emp_id_list = new ArrayList<>(all_emp_id_set);
        String emp_id = all_emp_id_list.get(int_rnd);

        EmpDao empDao =  empClientRedisTemplate.getEmpRedisData(emp_id);


        SavingMoneyEntity savingMoneyEntity = new SavingMoneyEntity(empDao.getEmp_id() , new Random().nextInt(10000));
        int saving_id = savingMoneyRepository.save(savingMoneyEntity).getSavingMoneyId();

        Optional<SavingMoneyEntity> selectSavingMoneyEntity = savingMoneyRepository.findById(saving_id);
        if(selectSavingMoneyEntity.isPresent()){
            SavingMoneyEntity entity = selectSavingMoneyEntity.get();
            return new ResponseEntity<>(new SavingMoneyDao(entity.getSavingMoneyId() , entity.getEmp_id(), empDao.getEmp_name() , entity.getMoneyAmt()) , HttpStatus.OK);
        }

            return new ResponseEntity<>(null , null, HttpStatus.OK);
    }catch(Exception e){
        log.error(e.toString());
        return new ResponseEntity<>(null , null , HttpStatus.OK);
    }
}

```
이제 핵심코드 입니다 지난시간까지는 계속해서 empClient 와 지속적인 통신을 해서 emp 데이터를 가져와서 해당 로직을 진행을 했다면 이제는 자신이 가지고 있는 redisDB 의 캐싱을 통해서 
진행을 하게 됩니다 이렇게 만들게 되면 SavingMoney 모듈은 empClient 와 강한 결합에서 느슨한 결합으로 바뀌게 됩니다 empClient 서버 상태의 유무에 상관 없이 SavingMoney 는 기존의 캐싱 DB 를 활용해서 지속적으로 서비스를 이어나갈 수 있게 되는 것입니다 

우리는 지난시간에서 부터 이번시간까지 비동기 메세지를 이용해서 분산 캐싱에 대해서 알아보았습니다 




