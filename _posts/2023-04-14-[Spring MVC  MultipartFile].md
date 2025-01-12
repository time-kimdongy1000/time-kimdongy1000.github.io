---
title: Spring MVC MultipartFile
author: kimdongy1000
date: 2023-04-14 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC , MultipartFile ]
math: true
mermaid: true
---

## MultipartFile
MultipartFile는 스프링에서 파일을 업로드할 때 사용하는 클래스 이름입니다 이 클래스는 인터페이스로 클라이언트에서 넘어오는 파일들을 관리할 수 있습니다

## 사전작업
maven
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

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-jpa -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-jdbc -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/com.h2database/h2 -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.1.214</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.24</version>
    <scope>provided</scope>
</dependency>

```
참고로 lombok는 getter , setter 자동으로 만들어줘서 코드의 길이를 줄여주는 것으로 앞으로 계속 채용하겠습니다

이번 시간에는 web에서 화면을 만들어서 사용을 진행을 하겠습니다 업로드 전반적인 내용, 다운로드 전반적인 내용 전부를 다룰 예정입니다
코드가 엄청 많을 예정이다 업로드부터 다운로드까지 이 한 페이지로 만들 예정 설명은 최대한 업로드, 다운로드 관점인 MuiltPartFile에 대해서만 다룰 예정입니다

## 업로드 

uploadFile.html
```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>파일 업로드</title>
</head>
<body>

    <form id = "file-form" method = "POST" enctype="multipart/form-data" action="/uploadFile">
        File: <input id = "file-form-input" type="file" multiple="multiple" name="file">
        <input id = "file-form-button" type="button" value="Upload"/>
    </form>



<script src = "/js/file.js"></script>
</body>
</html>

```

```

const file_form = document.querySelector("#file-form");
const file_form_input = document.querySelector("#file-form-input");
const file_form_button = document.querySelector("#file-form-button");

const allow_type = ["image/jpeg" , "image/png" , "image/jpg" , "text/plain"]
const allow_ext = ["txt" , "jpg" , "png" , "jpeg"]

function file_test(){

     const selectedFile = file_form_input.files[0];
     let return_flag = true;

     if(!selectedFile){
        return false;
     }

     console.log(selectedFile);
     const file_name = selectedFile.name;
     const file_ext = file_name.split(".")[1];
     const file_type = selectedFile.type;

     console.log(file_name , file_ext , file_type);

    if(!allow_ext.includes(file_ext)){
        alert('허용되지 않은 확장자입니다.')
        file_form_input.value = "";
        return false ;
    }

    if(!allow_type.includes(file_type)){
        alert('허용되지 않은 파일 타입입니다.')
        file_form_input.value = "";
        return false ;
    }

    return return_flag
}



file_form_input.addEventListener('change' , (e) => {

    file_test();

});

file_form_button.addEventListener('click' , (e) => {

    const file_test_flag = file_test();

    if(!file_test_flag){
        alert('파일업로드 테스트 통과에 실패했습니다.')
        return
    }
    file_form.submit();

})


```

UploadFileController.java
```

@Autowired
private UploadFileService uploadFileService;

@GetMapping("/uploadFile")
public String filePage(){

    return "uploadFile";
}



@PostMapping("/uploadFile")
public String uploadFile(@RequestParam("file") MultipartFile multipartFile){

    try{

        uploadFileService.uploadFile(multipartFile);

        return "redirect:/fileDown";

    }catch(Exception e){
        throw new RuntimeException(e);
    }
}


```

UploadFileController 핸들러는 2가지가 존재하는데 GetMapping는 페이지 요청 핸들러이고 PostMapping 은 파일 업로드 핸들러입니다
이때 파일 업로드 핸들러를 보면 `@RequestParam("file") MultipartFile multipartFile` 가 있습니다 이것이 앞의 form에서
`File: <input id = "file-form-input" type="file" multiple="multiple" name="file">`에서 이름 name의 key 값으로 가져오는 파라미터입니다


UploadFileService.java
```

package com.cybb.main.service;

@Service
@RequestScope
public class UploadFileService {

    private static final String[] allowExt = {"txt" , "jpg" , "png" , "jpeg"};
    private static final String[] allowContentType = {"image/jpeg" , "image/png" , "image/jpg" , "text/plain"};
    private static final String baseDir = "C:\\Users\\kimdongy1000\\Desktop\\spring_test";

    @Autowired
    private FileRepository fileRepository;


    public void uploadFile(MultipartFile multipartFile) throws Exception{

        String originFilaName = multipartFile.getOriginalFilename();
        String contentType = multipartFile.getContentType();
        String extName = originFilaName.split("\\.")[1];

        boolean extFlag = false;
        boolean contentTypeFlag = false;


        for(int i = 0; i < allowExt.length; i++){
            if(allowExt[i].equals(extName)){
                extFlag = true;
                break;
            }
        }

        for(int i = 0; i < allowContentType.length; i++){
            if(allowContentType[i].equals(contentType)){
                contentTypeFlag = true;
                break;
            }
        }

        if(!extFlag){
            throw new RuntimeException("허용된 확장자가 아닙니다.");
        }

        if(!contentTypeFlag){
            throw new RuntimeException("허용된 content-type 이 아닙니다");
        }

        Date date = new Date();
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyyMMdd");
        String str_today = simpleDateFormat.format(date);

        File file = new File(baseDir + "\\" + str_today);

        if(!file.exists()){
            file.mkdirs();
        }

        String randomFileName = UUID.randomUUID().toString();

        FileOutputStream fos = new FileOutputStream(baseDir + "\\" + str_today + "\\" + randomFileName + "." + extName);
        byte [] upFileByte = multipartFile.getBytes();

        fos.write(upFileByte);
        fos.close();

        insertUploadFileEntity(originFilaName , randomFileName , baseDir + "\\" + str_today + "\\" + randomFileName + "." + extName);

    }

    private void insertUploadFileEntity(String originFilaName , String randomFileName , String filepath) throws Exception
    {
        FileEntity fileEntity = FileEntity.builder()
                .fileOriginName(originFilaName)
                .fileRanName(randomFileName)
                .filePath(filepath)
                .build();

        fileRepository.save(fileEntity);
    }
}


```
Service는 RequsestScope를 사용했습니다 혹시나 서비스 bean 을 동시 호출할 때 동시성 문제를 없앨 것입니다
Service 전체로직은 업로드 파일을 검증하고 로컬 장치에 저장을 한 뒤 그 정보 (원래 이름, path 등 ) 엔티티로 저장 전체 로직입니다

클라이언트에서 파일을 보낼 때에는 -> MultipartFile 사용하지만 결국 파일을 로컬 특정 위치에 생성하는 것은 FileOutputStream이다 이 OutputStream에 대해서는
한번 전체적으로 정리할 수 있을 테니 지금은 파일을 출력할 때 사용하는 api라고 생각하면 된다

## 업로드 핵심
```

File file = new File(baseDir + "\\" + str_today); // 저장할 디렉터리 지정

if(!file.exists()){ // 지정된 디렉터리 없으면 생성 (부모 폴더 전부 포함)
    file.mkdirs();
}


String randomFileName = UUID.randomUUID().toString(); // 랜덤 파일 이름 생성

FileOutputStream fos = new FileOutputStream(baseDir + "\\" + str_today + "\\" + randomFileName + "." + extName); // 파일을 실제로 출력할 위치 지정
byte [] upFileByte = multipartFile.getBytes();  // multipartFile 파일 byte 화 

fos.write(upFileByte); // 바이트 파일을 write
fos.close();    // FileOutputStream 자원 회수 

```
결국 파일 업로드의 핵심은 이 부분이다 결국 보면 파일이라는 것은 프로그래밍 언어로 쓰일 때 getBytes 배열 형태와 만든 다음 지정 위치에 getBytes 배열을 쓰는 것이 핵심이다


## 파일 다운로드
그럼 파일 다운로드는 어떻게 만들어지는지 보자 파일 다운로드 핵심은 먼저 파일의 위치 DB로 읽어서 보여주는 것이 우선이다 그럼 그 작업을 먼저 진행을 해보면
물론 다운로드에서는 MuiltpartFile 이 사용되지는 않는다


DownloadFileController.java
```
package com.cybb.main.controller;

@Controller
public class DownloadFileController {

    @Autowired
    private DownloadFileService downloadFileService;

    @Autowired
    private ResourceLoader resourceLoader;

    @GetMapping("/fileDown")
    public String fileDownPage(Model model){

        try{

            List<FileDao> fileList = downloadFileService.getDownFileAllList();

            model.addAttribute("fileList" , fileList);

            return "downFile";

        }catch (Exception e){
            throw new RuntimeException(e);
        }
    }

    @GetMapping("/fileDown/{file_id}")
    public ResponseEntity<?> fileDown (@PathVariable Long file_id)
    {

        try{

            File donwFile = downloadFileService.downFile(file_id);

            Resource resource = resourceLoader.getResource("file:" + donwFile.getPath());

            return ResponseEntity.ok().header(HttpHeaders.CONTENT_DISPOSITION, "attachement; filename=\"" + donwFile.getName() + "\"")

                    .header(HttpHeaders.CONTENT_LENGTH, String.valueOf(donwFile.length())).body(resource);



        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}

```

마찬가지로 2개의 핸들러가 필요하다 하나는 다운로드 페이지로 이동하는 핸들러 다른 하나는 파일 다운로드 핸들러이다 파일 다운로드는 거의 대부분 FileOutputStream
을 많이 사용하지만 이번 시간에는 Resource를 이용해서 보내는 방법을 채택할 것이다


DownloadFileService
```

package com.cybb.main.service;

@Service
public class DownloadFileService {

    @Autowired
    private FileRepository fileRepository;

    public List<FileDao> getDownFileAllList()  throws Exception{

        List<FileEntity> opFileEntity = fileRepository.findAll();



        return opFileEntity.stream().map(x -> {

            FileDao fileDao = new FileDao(x.getId() , x.getFileOriginName() , x.getFileRanName() , x.getFilePath());

            return fileDao;

        }).collect(Collectors.toList());
    }

    public File downFile(Long fileId) throws Exception{

        Optional<FileEntity> opFileEntity = fileRepository.findById(fileId);

        if(opFileEntity.isPresent()){

            FileEntity fileEntity =opFileEntity.get();


            File downFile = new File(fileEntity.getFilePath());

            return downFile;

        }else{
            throw new RuntimeException("해당되는 파일이 없습니다");
        }

    }
}


```
파일 다운로드 페이지로 이동할 때 table에 표기할 파일 리스트 가져오는 service 그리고 Long 타입으로 넘어오는 id를 기반으로 파일을 return 하는 서비스 두 개가 있다
이 둘을 이용해서 File 을 return 하면 이제 back-end 쪽에서 파일 다운로드 끝이 난다

그리고 html 을 살펴보면


```

<!DOCTYPE html>
<html lang="en">
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>파일 다운로드 페이지</title>
</head>
<body>

  파일 다운로드 페이지

  <table class="tb_col">
      <tr>
          <th>File Id</th>
          <th>File OriginName</th>
          <th>File RandomName</th>
          <th>File Path</th>
      </tr>
      <tr th:each="file : ${fileList}">
          <td th:text="${file.id}"></td>
          <td th:text="${file.fileOriginName}"></td>
          <td th:text="${file.fileRanName}"></td>
          <td th:text="${file.filePath}"></td>
          <td><button id = "btn_file_down" th:value="${file.id}"/>파일다운로드</button></td>
      </tr>
  </table>

<script src = "/js/filedown.js"></script>
</body>
</html>

```

여기서는 타임리프 문법을 사용할 것이다 th:each를 이용해서 넘겨받은 파일 리스트를 돌면서 테이블을 생성할 것이다 그리고 제일 끝에는 file_id를 가지고 있는 버튼을 생성할 것이다 저기 버튼을 누르게 되면 js 문법에서 herf 요청을 할 것인데

```

const file_down_button = document.querySelector("#btn_file_down");

file_down_button.addEventListener('click' , (e) => {

    const file_id = e.target.value;

    location.href = `/fileDown/${file_id}`;


})



```

이런 식으로 이루어져 있다 그러면 우리는 파일 업로드 다운로드 둘 다 구축을 해보았다 다음 장에는 fetch로만 구성해서 파일 업로드 다운로드를 구현해 볼 예정이다





