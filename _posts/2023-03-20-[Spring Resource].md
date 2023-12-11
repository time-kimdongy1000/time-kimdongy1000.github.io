---
title: Spring Ioc Resource
author: kimdongy1000
date: 2023-03-20 14:20
categories: [Back-end, Spring - Core]
tags: [ Resource ]
math: true
mermaid: true
---

## Resource
Spring 으로 넘어오면서 리소스 처리 방식에 대해서 변화가 생겼습니다 org.springframework.core.io. 하위에 있는 Resource 는 하위 수준 리소스에 대한 액세스를 더욱 쉽게 만들기 위해서 만들어진 인터페이스 입니다 아래에 있는 메서드 또는 필드를 보면 유추하기 쉬운 것들로 이루어져 있고 

이번시간에는 UrlResource , FileSystemResource , ClassPathResource





## UrlResource
이는 spring 프레임워크에서 java.net.URL 기반으로 하는 리소스를 나타내는 클래스입니다 이 클래스는 HTTP , HTTPS , FTP 도 쓰이긴 하지만 가장많이 쓰이는 URL 을 통해 접속이 가능한 
리소스를 처리하는데 사용됩니다 

```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {


	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		URL url = new URL("https://mblogthumb-phinf.pstatic.net/MjAyMTAyMjJfMTM3/MDAxNjEzOTk1Nzg4MzE1.XUHyWN0J0DfNb1PN3LwqjnrzYYdC31UUIOLlXd9esgog.7ZSWKBEc4zXdqF_zsA9iRVZEQZLnCPfbuT1uUznpWLUg.JPEG.dangoon123/Divergence_Meter_0.409031.jpg?type=w420");
		Resource urlResource = new UrlResource(url);

		if(urlResource.exists()){
			System.out.println("URL Resource File exists!");
		}
	}
}

```

지금보면 https 주소로 URL 객체를 만들고 그 객체를 Resource 타입의 객체로 만들어서 그 파일이 존재하냐 안하냐로 지금 if 문을 통과 하고 있습니다 단 Resource 특정상 
파일을 읽어올 수는 있는데 다운로드 할 수 있는 그런 인터페이스는 아니다 보니 지금처럼 사진을 받는 그런 리소스같은 경우는 부적절합니다 그리고 매 사용마다 새로운 Resoucre 객체를 생성하는 것을 볼 수 있습니다 그래서 Spring 은 하나의 bean 으로 모든 리소스를 처리할 수 있는 인터페이스를 하나 더 만들었는데 그것이 바로 ResourceLoader 입니다 
그렇다고 사진다운로드를 못하는게 아닙니다 적절한 다른 FileStream 을 사용하면되는데 그 주제는 




## ResourceLoader
ResourceLoader 특징은 계속 언급은 하겠지만 자신이 읽어올 파일에 맞는 Resoucre 를 구현해서 사용해야 합니다 지금같이 HTTP 통신에서 Resoucre 를 가져올때는 URL 리소스를 
반드시 써여 하는데 ResourceLoader 특징은 앞에 적어주는 prefix 를 분석해서 적절한 Resoucre 객체의 타입을 만들어내게 됩니다 

```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {
	
	@Autowired
	private ResourceLoader resourceLoader;

	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		Resource urlResource = resourceLoader.getResource("https://mblogthumb-phinf.pstatic.net/MjAyMTAyMjJfMTM3/MDAxNjEzOTk1Nzg4MzE1.XUHyWN0J0DfNb1PN3LwqjnrzYYdC31UUIOLlXd9esgog.7ZSWKBEc4zXdqF_zsA9iRVZEQZLnCPfbuT1uUznpWLUg.JPEG.dangoon123/Divergence_Meter_0.409031.jpg?type=w420");

		if(urlResource.exists()){
			System.out.println("URL Resource File exists!");
		}
	}
}
```

지금보면 ResourceLoader 주입을 받고 그 빈을 통해서 getResource 를 호출하게 되면 우리는 아까처럼 new UrlResource 를 만들지 않고도 자동으로 spring 이 해당 url 을 분석해서 
적절한 bean 타입을 맞춰주게 됩니다 이렇게 될 수 있는 원리가 무엇이냐면 앞에 적었지만 prefix 를 적어주게 되는데 http , https 통신은 앞에 프로토콜을 적어줌으로 이게 
http 통신이니 UrlResource 객체를 만들어줘 하고 스프링에게 전가를 시키는것입니다 그럼 스프링은 알아서 그 타입에 맞게 Resoucre를 만들어주게 됩니다 


## UrlResouceLoader 로 사진 파일 다운로드 
```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {

	@Autowired
	private ResourceLoader resourceLoader;


	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		Resource urlResource = resourceLoader.getResource("https://mblogthumb-phinf.pstatic.net/MjAyMTAyMjJfMTM3/MDAxNjEzOTk1Nzg4MzE1.XUHyWN0J0DfNb1PN3LwqjnrzYYdC31UUIOLlXd9esgog.7ZSWKBEc4zXdqF_zsA9iRVZEQZLnCPfbuT1uUznpWLUg.JPEG.dangoon123/Divergence_Meter_0.409031.jpg?type=w420");

		if(urlResource.exists()){
			System.out.println("URL Resource File exists!");

			InputStream inputStream = urlResource.getInputStream();
			FileOutputStream outputStream = new FileOutputStream("C:\\Users\\kimdo\\OneDrive\\바탕 화면\\spring_test\\time_macine.png");

			byte[] buffer = new byte[10024];
			int bytesRead;
			while ((bytesRead = inputStream.read(buffer)) != -1) {
				outputStream.write(buffer, 0, bytesRead);
			}

			outputStream.close();
			inputStream.close();

		}
	}
}
```
추가적인 내용으로 우리는 우리 로컬에 웹상에 있는 사진을 다운로드 할 수 있습니다 


## ClassPathResource
이는 현재 어플리케이션 classPath 상에 있는 리소스를 읽는 역할을 하는 클래스입니다 어플리케이션 안에 파일을 하나 만들겠습니다 

-project 
	-src
		-main
			-java
				-com
					-cybb
						-main
							-SpringBootWebSecurityApplication.java 
						-resource
							-application.properties
							-dbList.txt

지금 같은 위치에 dbList.txt 파일을 하나 만들겠습니다

## dbList.txt
```
User
Dept
Item
```

## ClassPathResource 로 classPath상의 파일 읽기 시작 
```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {

	@Autowired
	private ResourceLoader resourceLoader;

	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		Resource classpathResource = resourceLoader.getResource("classpath:/dbList.txt");

		if(classpathResource.exists()){

			File file = classpathResource.getFile();
			InputStream fileInputStream = new FileInputStream(file);
			BufferedReader reader = new BufferedReader(new InputStreamReader(fileInputStream));

			String line;
			while( (line =  reader.readLine()) != null ){
				System.out.println(line);
			}
		}
	}
}


```
마찬가지로 이때는 prefix 가 앞에서는 https 였기 떄문에 자동으로 UrlResourceLoader 이 동작했다면 지금은 prefix 가 classPath 로 변경이 되었기 때문에 스프링은 자동으로 이 
Resource 객체를 ClasspathResource 로 변경을 하게 됩니다 그리고 파일을 읽는거 까지 읽어서 한줄한줄 표현까지 진행을 했습니다 

## FileSystemResource
```
@SpringBootApplication
public class SpringBootWebSecurityApplication implements ApplicationRunner {

	@Autowired
	private ResourceLoader resourceLoader;


	public static void main(String[] args) {
		SpringApplication.run(SpringBootWebSecurityApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		Resource filepathResource1 = resourceLoader.getResource("file:/home/kimdongy1000/springProject/config/mysql_db.properties");
		Resource filepathResource2 = resourceLoader.getResource("file:/home/kimdongy1000/springProject/config/oracle_db.properties");

		if(filepathResource1.exists()) {

			File file = filepathResource1.getFile();
			InputStream fileInputStream = new FileInputStream(file);
			BufferedReader reader = new BufferedReader(new InputStreamReader(fileInputStream));

			String line;
			while ((line = reader.readLine()) != null) {
				System.out.println(line);
			}
		}else{
			System.out.println("file1 존재하지 않습니다");
		}

		if(filepathResource2.exists()) {

			File file = filepathResource2.getFile();
			InputStream fileInputStream = new FileInputStream(file);
			BufferedReader reader = new BufferedReader(new InputStreamReader(fileInputStream));

			String line;
			while ((line = reader.readLine()) != null) {
				System.out.println(line);
			}
		}else{
			System.out.println("file2 존재하지 않습니다");
		}
	}
}

```
FileSystemResource 말그대로 파일시스템의 위치에서 접근하는것이다 ClassPathResource classPath 안에서만 움직이기 때문에 외부파일에 접근하는게 약간의 한계가 있지만 
FileSystemResource 리소스 같은 경우는 외부 파일을 엑스만 할 수 있다면 ClassPathResource 보다 강력하게 사용할 수 있습니다 이때 prefix 는 file 이라는 prefix 를 사용해서 
파일위치를 명시를 해주고 있습니다 

오늘은 이렇게 해서 spring 다양한 리소스 관리에 대해서 공부를 해보았습니다