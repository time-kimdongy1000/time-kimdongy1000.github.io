---
title: JAVA  Files_walkFileTree
author: kimdongy1000
date: 2022-01-01 10:00
categories: [Back-end, Java]
tags: [Files_walkFileTree]
math: true
mermaid: true
---

## Files.walkFileTree
실제로 사용할 때는 Files.walkFileTree로 사용하며 이는 JAVA NIO(Non-blocking I/O) 패키지에서 제공하는 파일 및 디렉터리 트리를 순회하면서 파일이나 디렉터리에 대한 작업을 수행할 수 있습니다

```
public class Practices01 {

	/* window 기준 */
	private static final String ROOT_PATH = "C:\\webProject\\jenkins_practies";

	public static void main(String[] args) {

		try {

			Files.walkFileTree(Path.of(ROOT_PATH), new HashSet<>(), Integer.MAX_VALUE, new FileVisitor<Path>() {

				@Override
				public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
					System.out.println("preVisitDirectory: " + dir);
					return FileVisitResult.CONTINUE;
				}

				@Override
				public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
					System.out.println("visitFile: " + file);
					return FileVisitResult.CONTINUE;
				}

				@Override
				public FileVisitResult visitFileFailed(Path file, IOException exc) throws IOException {
					System.out.println("visitFileFailed: " + file);
					return FileVisitResult.CONTINUE;
				}

				@Override
				public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
					System.out.println("postVisitDirectory: " + dir);
					return FileVisitResult.CONTINUE;
				}
			});

		} catch (Exception e) {
			throw new RuntimeException(e);
		}

	}

}
```

## 그럼 지금 볼려고 하는 디렉터리 구성은 어떠한가?
jenkins_practies
	-jobs
		-jenkins_module1
			-builds
				-1
					-workflow
						-2.xml
						-3.xml
						-4.xml
						...
					-build.xml
					-log
					-log-index
				-2
					-workflow
						-2.xml
						-3.xml
						-4.xml
						...
					-build.xml
					-log
					-log-index
				-legacyIds
				-permalinks
			-config.xml
			-nextBuildNumber

지금 디렉터리 구조는 실제 사용되고 있는 구조이다 jenkins에서 jobs를 들어가 보면 이런 형식으로 로그 파일 및 빌드 파일을 관리하고 있는데 좋은 예제가 될 거 같아서 가지고 있는 중이다
그럼 저 구조는 저렇다 치고 먼저 실행을 해보자

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/ec2c749e-af44-4471-9adb-a7616e5a6ab4)

## 기본적인 원리 

`Files.walkFileTree(Path.of(ROOT_PATH), new HashSet<>(), Integer.MAX_VALUE, new FileVisitor<Path>()`를 살펴보면
Path.of는 순회할 위치를 먼저 지정하게 됩니다
Integer.MAX_VALUE 같은 경우는 이제 순회 디렉터리 depth를 설정하게 되는데 Integer.MAX_VALUE는 거의 모든 디렉터리를 순회한다고 생각하면 되고 1 , 2로 적혀 있으면
0은 자기 자신만 순회  1은 자기 자신 + 자기 바로 아래 디렉터리까지만 순회를 한다는 뜻입니다
`new FileVisitor<Path>()` 방문할 때 사용할 함수입니다


preVisitDirectory , postVisitDirectory는 모두 디렉터리에서 작업을 행하는 애들이다 이 둘의 특징은 시점인데 preVisitDirectory는 디렉터를 방문할 때 호출되는 콜백 함수이고 postVisitDirectory 디렉터리를 떠날 때 호출되는 콜백 함수이다 visitFile , visitFileFailed는 디렉터리 상에서 파일을 맞추질 때 작업을 행하는 콜백 함수이다 이름에도 알 수 있다시피
파일을 올바르게 읽었으면 visitFile 콜백 될 것이고 그렇지 않으면 visitFileFailed 이 호출됩니다

방문하는 순서는 다음과 같습니다 

1. 폴더 우선순위입니다 폴더가 있으면 폴더를 먼저 순회하려고 합니다

2. 폴더와 파일이 같이 있으면 파일을 먼저 순회합니다
jenkins_practies
	-jobs
		-jenkins_module1
			-builds
				-1
					-workflow
						-2.xml
						-3.xml
						-4.xml
						...
					-build.xml
					-log
					-log-index
				-2


이렇게 디렉터리 위치가 있을 때에는

jenkins_practies -> jobs -> jenkins_module1 -> builds -> 1  여기까지는 폴더 우선순위로 방문하는 것입나다 물론 폴더밖에 없으니까 당연하긴 하지만

1번 디렉터리 안에는 workflow 폴더와 그 아래 숫자로 된 xml 파일과, build.xml 파일이 존재하고 있습니다 이때는 폴더 우선순위가 아니라 파일 우선순위로 순위가 변경이 됩니다

jenkins_practies -> jobs -> jenkins_module1 -> builds -> 1 -> build.xml -> log -> iog-index 이렇게 끝나면 다시 폴더 우선순위로 변경이 됩니다

여기까지만 봐도 이 소스의 대략적인 설명을 알 수 있을 텐데 구체적으로 로직 몇 가지를 넣어보도록 하겠습니다


1. 디렉터리를 순회하면서 확장자가 xml로 끝나는 파일의 개수를 세려고 할 때 다음과 같은 로직을 추가합니다
```
public class Practices01 {

	/* window 기준 */
	private static final String ROOT_PATH = "C:\\webProject\\jenkins_practies";
	
	static int end_with_xml_cnt = 0;

	public static void main(String[] args) {

		try {
	

			Files.walkFileTree(Path.of(ROOT_PATH), new HashSet<>(), Integer.MAX_VALUE, new FileVisitor<Path>() {

			
				@Override
				public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
					System.out.println("visitFile: " + file);
					
					if(file.toString().toLowerCase().endsWith(".xml")) {
						end_with_xml_cnt++;
					}
					return FileVisitResult.CONTINUE;
				}

			});
			
			System.out.print(end_with_xml_cnt);

		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
}
```
상단에 `static int end_with_xml_cnt = 0;`를 넣고 visitFile에서 로직을 구현을 해주면 된다

마찬가지로 폴더도 해보자 폴더의 이름이 숫자로 된 것 중에서 홀수인 폴더의 개수


```
public class Practices01 {

	/* window 기준 */
	private static final String ROOT_PATH = "C:\\webProject\\jenkins_practies";
	
	static int end_with_xml_cnt = 0;
	static int odd_folder_name_cnt = 0;
	
	public static boolean isNumeric(String str){
        for(char ch : str.toCharArray()){
            if(!Character.isDigit(ch)){
                return false;
            }
        }
        return true;
    }

	public static void main(String[] args) {

		try {
			
			Files.walkFileTree(Path.of(ROOT_PATH), new HashSet<>(), Integer.MAX_VALUE, new FileVisitor<Path>() {

				@Override
				public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
					System.out.println("preVisitDirectory: " + dir);
					
					if(isNumeric(dir.getFileName().toString())) {
						
						int file_name_num = Integer.parseInt(dir.getFileName().toString());
						if(file_name_num % 2 == 1) {
							odd_folder_name_cnt++;
						}
						
					}
									
					return FileVisitResult.CONTINUE;
				})

			System.out.println(odd_folder_name_cnt);

		} catch (Exception e) {
			throw new RuntimeException(e);
		}

	}

}


```
이렇게 구할 수 있습니다 오늘은 java 패키지 중에서 폴더 및 파일을 순회하는 패키지인 Files.walkFileTree 대해서 공부를 해보았습니다 







