---
title: Spring MicroService 21 Spring MicroService 미니프로젝트
author: kimdongy1000
date: 2023-08-06 10:00
categories: [Back-end, Spring - MicroService]
tags: [Spring - MicroService]
math: true
mermaid: true
---

## 미니프로젝트
이번시간에는 앞에서 배운것들을 총망라해서 미니프로젝트를 한번 만들어보겠습니다 이제까지 배운것을 전부 다 쓰지는 않겠지만 그래도 각각의 핵심 파트건들을 사용해서 간단한 curd 어플리케이션을 만들어보겠습니다 

## 프로젝트 구조

이 프로젝트는 empClient 와 savingMoney 인스턴스를 만들어서 empCliet 에서 emp 를 만들고 조회하는 인스턴스 savingMoney 는 empClient 에 속하는 emp 에 대해서 돈을 적립하는 
프로세서입니다

## 전체소스
<https://gitlab.com/kimdongy1000/spring-cloud-project/-/tree/main-mini-project?ref_type=heads>

전체소스에는 docker 를 포함한 docker-compose 까지 만들어서 바로 사용할 수 있게 개발이 되어 있습니다 

## 핵심코드
이번 미니프로젝트의 핵심은 화면까지 만들어서 기동을 했고 화면에 필요한 리소스 및 api 요청은 gateway 인스턴스에서 진행을 했으며 핵심코드를 같이 보면서 진행을 하겠습니다 

## miniProjectGateWay - createRandomMoney
```

@PostMapping("/createRandomMoney")
public ResponseEntity<?> createRandomMoney()
{
    try{

        String serviceUrl = searchEurekaService.getInstancesUri("miniProject-EmpClient");
        RestTemplate restTemplate = new RestTemplate();

        //Random emp 객체 가져오기
        String requestUrl = String.format("%s/readRandomEmp", serviceUrl);

        RequestEntity<Void> requestEntity = RequestEntity
                .get(requestUrl)
                .header("application/json")
                .build();

        ResponseEntity<EmpDao> response = restTemplate.exchange(requestEntity, EmpDao.class);

        //Random emp 객체에 랜던함 금액 설정
        String serviceUrl2 = searchEurekaService.getInstancesUri("miniProject-SavingMoney");

        String requestUrl2 = String.format("%s/createMoney", serviceUrl2);

        RequestEntity<EmpDao> requestEntity2 = RequestEntity
                .post(requestUrl2)
                .header("application/json")
                .body(Objects.requireNonNull(response.getBody()));


        ResponseEntity<SavingMoneyDao> response2 = restTemplate.exchange(requestEntity2, SavingMoneyDao.class);

        //Random emp 객체에 랜던함 금액값 가져와서 return
        String requestUrl3 = String.format("%s/readEmp/%s", serviceUrl , Objects.requireNonNull(response2.getBody()).getEmp_id());

        RequestEntity<Void> requestEntity3 = RequestEntity
                .get(requestUrl3)
                .header("application/json")
                .build();

        ResponseEntity<EmpDao> response3 = restTemplate.exchange(requestEntity3, EmpDao.class);

        Map<String , Object> resultMap = new HashMap<>();
        resultMap.put("emp_id", Objects.requireNonNull(response3.getBody()).getEmp_id());
        resultMap.put("emp_name", Objects.requireNonNull(response3.getBody()).getEmp_name());
        resultMap.put("moneyAmt", Objects.requireNonNull(response2.getBody()).getMoneyAmt());

        Gson gson = new Gson();
        String resultJson = gson.toJson(resultMap, resultMap.getClass());



        return new ResponseEntity<>(resultJson , HttpStatus.OK);
    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

```
해당 메서드는 총 3번 하위 인스턴스ㅗ와 통신합니다 첫번째 통신은 miniProject-EmpClient 에서 랜덤한 emp_id 를 가져옵니다 두번째 통신은 miniProject-SavingMoney 에 랜덤한 emp_id 를 보내고 이곳에서 해당 emp_id 에 대한 금액적립 api 를 사용해서 데이터를 저장합니다 세번째 통신은 랜덤한 유저의 정보를 가져와서 랜덤한 금액을 조합해서 return 하는 것입니다 사실 세번째 통신은 굳이 필요없지만 정합성 판별을 위해서 한번더 통신을 하게끔 만들었습니다 


오늘은 이렇게 간단하게 미니프로젝트를만들어보았습니다 해당 미니 프로젝트를 이용해서 다음시간에 활용할 oauth2 프로젝트로 활용을 할 예정입니다









