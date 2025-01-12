---
title: Spring MVC MultipartFile2
author: kimdongy1000
date: 2023-04-15 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC , MultipartFile ]
math: true
mermaid: true
---

## MultipartFile
우리는 앞에서 form 요청으로 파일 업로드 다운로드를 전부 만들었습니다 이번 시간에는 back-end 없이 비동기 fetch로 파일 업로드 다운로드를 한번 구축해 보겠습니다
Front - end만 수정


## file.js 
결국 기존의 로직 그대로 변경이 되는 것이고 요청하는 방법만 달라지게 되는 것이다 그럼 upload부터 알아보자

```

file_form_button.addEventListener('click' , (e) => {

    const file_test_flag = file_test();

    if(!file_test_flag){
        alert('파일업로드 테스트 통과에 실패했습니다.')
        return
    }
    //file_form.submit();

    let fileForm = new FormData();
    fileForm.append("file" , file_form_input.files[0]);

    fetch("http://localhost:8080/uploadFile" , {
        method : "POST" ,
        body : fileForm

    }).then((response) => {
       const statusCode = response.status;

       if(statusCode === 200){
            alert('파일업로드가 완료 되었습니다.');
       }
    })
    .catch((error) => {
        console.error(error)
    })

})


```

기존 file_form.submit();에서 이 부분이 주석이 잡혀 있고 fetch 가 들어왔다 이때 중요한 것은 fetch는 이 화면에서 응답 값은 clinet 요청입니다
그렇게 되면 이 핸들러들은 실제로 ResponseBody를 주저야 함으로 이에 대한 설정을 변경하겠습니다

```

@PostMapping("/uploadFile")
public ResponseEntity<?> uploadFile(@RequestParam("file") MultipartFile multipartFile){

    try{

        uploadFileService.uploadFile(multipartFile);

        return ResponseEntity.ok().body(null);

    }catch(Exception e){
        throw new RuntimeException(e);
    }
}


```
기존의 uploadFile 핸들러레서 String으로 redirect 하는 것을 제외하고 그냥 단순 void로 바꾸어서 성공하면 catch로 빠지지만 않게 하겠습니다
이때는 ResponseEntity를 써도 되고 @ReposeBody를 사용해도 된다

자 그럼 파일 다운로드는 조금 복잡하다 애초에 ajax 나 fetch는 기본적으로는 파일 다운로드를 지원하지 않는다
그래서 blob로 return으로 넘어오는 것은 아래처럼 임시 a 태그를 만들어서 임시 path에 요청을 하는 것이다 다음처럼 말이다

```

file_down_button.addEventListener('click' , (e) => {

    const file_id = e.target.value;


    //location.href = `/fileDown/${file_id}`;

    fetch("http://localhost:8080/fileDown/" + file_id  , {
        method  : "GET"

    }).then((res) => {

        const headers = res.headers;
        filename = headers.get('Content-Disposition');

        return Promise.all([res.blob(), filename]);
    }).then(([body , filename]) => {
        f_name = filename.substring(filename.indexOf('"') +1 , filename.lastIndexOf('"'));

        const file_down_path = window.URL.createObjectURL(body);
        const a = document.createElement('a');
        a.href =  file_down_path;
        a.download = f_name

        document.body.appendChild(a);
        a.click();
        a.remove();

    })

})

```

이렇게 말이다 없는 a 태그를 만들어서 클릭을 하게끔 하고 다시 임시 a 태그를 지우는 것으로 파일 다운로드가 일어나게 된다
이때 blob 안에 이미 파일 다운로드와 관련된 데이터는 모두 들어 있기 때문에 임시 주소를 만들어서 재요청하는 것이다