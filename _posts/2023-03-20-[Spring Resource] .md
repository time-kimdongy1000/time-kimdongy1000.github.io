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
Spring 으로 넘어오면서 리소스 처리 방식에 대해서 변화가 생겼습니다 
org.springframework.core.io. 하위에 있는 Resource 는 하위 수준 리소스에 대한 액세스를 더욱 쉽게 만들기 위해서 만들어진 인터페이스 입니다 
아래에 있는 메서드 또는 필드를 보면 유추하기 쉬운 것들로 이루어져 있고 

기본적으로 java 의 파일 클래스를 다우러 보았다면 더욱 쉽게 보일것입니다 



interface Resource
```

package org.springframework.core.io;

public interface Resource extends InputStreamSource {

    boolean exists();

    boolean isReadable();

    boolean isOpen();

    boolean isFile();

    URL getURL() throws IOException;

    URI getURI() throws IOException;

    File getFile() throws IOException;

    ReadableByteChannel readableChannel() throws IOException;

    long contentLength() throws IOException;

    long lastModified() throws IOException;

    Resource createRelative(String relativePath) throws IOException;

    String getFilename();

    String getDescription();
}


```

## UrlResource file

일반적으로 URL 로 엑세스할 수 있는 모든 개체에 액세스 할때 사용할 수 있습니다  파일 시스템 , https , ftp classpath 등 다양한 곳에 접근할때 사용할 수 있습니다 
다만 new 를 사용해서 만들때에는 모두 한가지 객체로만 만들어서 Resource 의 유연성은 다소 떨어지는것을 볼 수 있습니다



```

package com.cybb.main;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.PropertySource;
import org.springframework.context.annotation.PropertySources;
import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;

import java.io.File;
import java.io.FileReader;

@SpringBootApplication
public class SpringRestartApplication implements ApplicationRunner {

	public static void main(String[] args) {
		SpringApplication.run(SpringRestartApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {

		Resource resource = new UrlResource("file:C:\\Users\\kimdongy1000\\Desktop\\spring_test\\test.txt");
		if(resource.exists()){
			System.out.println("파일이 존재합니다.");
			String filename = resource.getFilename();
			System.out.println("filename : " + filename);

			File file = resource.getFile();
			FileReader reader = new FileReader(file);

			int ch;
			while ((ch = reader.read()) != -1) {
				System.out.print((char) ch);
			}
		}
	}
}

```

우리는 앞에서 특별한 prefix 를 넣을 수 있다고 했다 준비물은 바탕화면에 간단한 메모장 준비하고 돌리자 결과는 이렇게 나올것이다 
이게 기본적으로 파일시스템에 접근해서 로컬 파일을 읽을때 (사실 prefix 를 줘도 되고 안주어도 알아서 읽어냅니다)

```

파일이 존재합니다.
filename : test.txt
테스트입니다

```

다만 이 절대경로에 있는것은 문제가 생긴다 지금 프로젝트가 jar 로 변환되어서 다른곳에 옮겨저서 배포 되면 현재 이 path 는 쓸 수 없는 로직이기 때문에 절대경로는 잘 사용하지 않습니다
당장 이 jar , war 패키징되어 리눅스 시스템에 던져지면 동작하지 않을것이기 때문이다 



## UrlResource https

```
@Override
public void run(ApplicationArguments args) throws Exception {

	Resource resource = new UrlResource("https://ichef.bbci.co.uk/news/800/cpsprodpb/E172/production/_126241775_getty_cats.png");
	if(resource.exists()){
		System.out.println("리소스 존재합니다.");
		
		InputStream inputStream = resource.getInputStream();
		FileOutputStream outputStream = new FileOutputStream("C:\\Users\\kimdongy1000\\Desktop\\spring_test\\sample_cat.png");

		byte[] buffer = new byte[10024];
		int bytesRead;
		while ((bytesRead = inputStream.read(buffer)) != -1) {
			outputStream.write(buffer, 0, bytesRead);
		}

		outputStream.close();
		inputStream.close();

	}else{
		System.out.println("리소스 존재하지 않습니다.");
	}



}

```
이번엔 특정 웹주소에 있는 사진을 한번 로컬로 다운로드를 해보자 지금 이 소스를 이용하면 해당 url 에 표현되는 사진을 나의 로컬 컴퓨터로 다운로드 받을 수 있습니다 

1. 웹에서 읽어온 파일을 먼저 InputStream 넣습니다 
2. FileOutputStream 를 이용해서 어디에 저장을 할것인지 위치를 명시합니다 
3. 위치 명시가 시작되면 거대한 byte 배열을 만들고 inputStream 으로 읽어내기 시작합니다 그것을 outputStream 써 내려갑니다
4. 그러다 더 읽을 소스가 byte 가 없으면 그대로 종료됩니다 


## UrlResource classPath

```
@Override
public void run(ApplicationArguments args) throws Exception {

	Resource resource = new UrlResource(new URL("classpath:/schema/User.sql").openConnection().getURL());
	System.out.println(new URL("classpath:/schema/User.sql").openConnection().getURL().toString());
	if(resource.exists()){
		System.out.println("리소스 존재합니다.");

		File file = resource.getFile();
		FileReader reader = new FileReader(file);

		int ch;
		while ((ch = reader.read()) != -1) {
			System.out.print((char) ch);
		}




	}else{
		System.out.println("리소스 존재하지 않습니다.");
	}



}


```

classpth:/schema/User.sql
```
Create Table User(
    name varchar(100) ,
    age varchar(100)
)

```

UrlResource 의 가장 큰 단점은 모든 것을 절대경로 즉 file 시스템으로 위치를 확인할려는 문제점이 있다 지금도 저기 결과를 찍어보면

```

file:/C:/Users/kimdongy1000/Documents/workspace-spring-tool-suite-4-4.11.0.RELEASE/Spring_Restart/target/classes/schema/User.sql
리소스 존재합니다.
Create Table User(
    name varchar(100) ,
    age varchar(100)
)


```

내가 classPath 주소로 명시는 해두었지만 결국 열어볼때는 file 이라는 접두사를 써서 파일 시스템으로 읽고 있는 모습니다 
그런 단점을 없애기 위해서 Srping ResourceLoader 를 활용해서 path 를 읽고 적절한 Resource 하위 클래스 객체를 생성하게 됩니다


## ResourceLoader

```

public interface ResourceLoader {

    Resource getResource(String location);

    ClassLoader getClassLoader();
}

```

모든 어플리케이션 컨텍스트는 ResourceLoader 를 활용해서 Resource 의 인스턴스를 얻을 수 있는데 아까 했던 모든 작업을 ResourceLoader 에 집어넣고 진행을 해보자
그리고 코드 하나를 더 추가해보자 `System.out.println(resource.getClass());`
이것을 추가하면 이제 resource 객체가 어떤 객체로 만들어지는지 나올것이다 

```

@Override
public void run(ApplicationArguments args) throws Exception {

	Resource resource = resourceLoader.getResource("classpath:/schema/User.sql");
	System.out.println(resource.getClass());
	
	if(resource.exists()){
		System.out.println("리소스 존재합니다.");

		File file = resource.getFile();
		FileReader reader = new FileReader(file);

		int ch;
		while ((ch = reader.read()) != -1) {
			System.out.print((char) ch);
		}




	}else{
		System.out.println("리소스 존재하지 않습니다.");
	}
}

```

```

class org.springframework.core.io.ClassPathResource
파일이 존재합니다.
filename : User.sql
Create Table User(
    name varchar(100) ,
    age varchar(100)
)

```
첫번째 Resource 객체는 ClassPathResource 가 나왔다 


## resourceLoader http 파일 요청

```
Resource resource = resourceLoader.getResource("https://ichef.bbci.co.uk/news/800/cpsprodpb/E172/production/_126241775_getty_cats.png");

```

```
class org.springframework.core.io.UrlResource
리소스 존재합니다.

```
http 요청일때는 UrlResource 변경되었다 


## resourceLoader 로컬 파일 시스템 접근 

```
Resource resource = resourceLoader.getResource("file:C:\\Users\\kimdongy1000\\Desktop\\spring_test\\test.txt");

```

```
class org.springframework.core.io.FileUrlResource
리소스 존재합니다.
테스트입니다

```

지금 보면 ResourceLoader 이 접두사를 보고 적절히 Resource 객체를 만들어내는것을 보았다 앞에서 사용한  new UrlResource 는 전부 한가지 객체만 만들어낼 수 밖에 없다 


```
class org.springframework.core.io.UrlResource

```

이제 spring 에서 외부 자원을 읽어올때는 ResourceLoader 를 사용해서 관리하는것이 좋습니다.













