---

title: DevOps Nexus 5 Proxy Repository
author: kimdongy1000
date: 2023-06-27 16:30
categories: [DevOps, Nexus]
tags: [ Nexus ]
math: true
mermaid: true

---

우리는 지난시간까지 해서 nexus 의 기능의 일부분을 공부했다 복습을 해보자면 

1. nexus 설치

2. 외부망이 허용된 상황에서 Centeral 저장소에 연결을 Proxy 연결을 해서 라이브러리 받아온것을 했고 

3. 나만의 그룹과 나만의 저장소를 만들어보았고 

4. 나만의 저장소에서 나만의 라이브러리를 만들어서 올리고 그것을 서로다른 프로젝트에 다운로드 받겠금

오늘 거의 이 글이 nexus 의 maven 파트에서 마지막이 될것입니다 나만의 저장소를 proxy 저장소로 땡겨오는것으로 마무리 짓겠습니다 2번과 비슷한 글이 될것긴 하지만 이 나름대로 의미가 있기에 진행을 하겠습니다 

이 실습을 위해서는 nexus 가 2개를 준비했습니다


![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/4c25105d-a8ed-4cbf-a8d2-e4af3260b5f3)

구조는 이렇게 될것입니다 그럼 정말 단계가 그냥 하나가 더 늘어난 모양인데 이게 정말 유효한 모양인가 생각할 수 있지만 실제로 이렇게 운영하는 사이트가 있습니다 

VM 머신 1번에서는 중앙하고 공통 개발자들이 라이브러리 개발후 1번 VM 머신에 넣어두고 2번 머신에서 애플리케이션 개발자들이 끌어다 쓰는 운영을 하기도 합니다 
그럼 이 모양으로 라이브러리를 받게 설정을 해보겠습니다



## Proxy repository 만들기

1. 상단 톱니바퀴 클릭후 왼쪽 메뉴 Repositories 클릭후 상단 메뉴의 create Repository 클릭 

2. 선택지 중에서 maven2 (proxy) 클릭 


![2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7f2be494-d743-481e-8a2c-700d6c18eea7)

상단의 타이틀은 원하는대로 적으시면됩니다 


그리고 아래에는 다음과 같은 선택지가 있는데 

What type of artifacts does this repository store?
Release 

릴리스는 아시다 싶히 snapshot 의 반대되는 개념이다 릴리스는 안정적인 버전으로 버그가 거의 없는 개발완료된 버전이고 snapshot 아직 개발이 진행중이고 배포를 통해서 다른 사람들로 부터 피드백을 받고자 할때 사용하는 버전이다 우리는 안정적인 버전만 받게끔 Releas 를 선책할것입니다


Validate that all paths are maven artifact or metadata paths
Permissive 

이 문장은 보안정책을 뜻하는것인데 아티펙트나 , 메타데이터가 유효한 path 인지 확인하는것인데 우리는 Permissive 로 좀 느슨한 보안정책을 활용할것이고 반대되는 개념인 strict 는 좀더 확실한 보안을 강구하고 규칙을 정할때 사용합니다

마지막으로 하나의 선택지 Content-Disposition  존재하지만 이는 기본값으로 두겠습니다 (먼지 모르겠습니다)

3. Remote storage 
이는 우리가 사용하는 VM 머신 1번의 group 주소를 proxy 를 해오겠습니다 


![3](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/59f8f078-7e1f-494d-82be-84dbb1e33497)

나머지도 크게 기본값에서 벗어나지는 않습니다 

Auto Bloacking enabled 
이를 체크하게 되면 원격피어가 응답하지 않거나 사용할 수 없는 상태가 된다면 바깥으로 나가는 연결을 자동으로 차단하는 것을 뜻합니다
이때 원격피어는 VM머신 1번 이 될것입니다 (group repository)

maximum component age 
이는 컴포넌트가 생명주기를 설정하는 곳입니다 -1을 기본값인데 이는 무제한으로 원격피어에서 받은 컴포넌트를 얼마나 오랫동안 보관할것인지 보관주기를 설정합니다 
만약 -1 이면 원격피어의 라이브러리가 변경이 되더라도 proxy 리포지토리는 동기화 되지 않습니다 (이렇게 -1 이 기본값인 이유는 릴리스는 안정된 버전으로 변경될 이유가 없다고 생각을 하기 때문입니다) 추천하는 대로 -1 을 써도 되지만 릴리스가 주기적으로 변경되는 (?) 곳들은 수치를 변경해도 됩니다 
물론 릴리스가 변경되어서 자주 배포되는것은 릴리스가 될 수 없다는 점도 알고 있어야 합니다 

하단도 마찬가지로 컴포넌트의 metadata age 를 설정하는곳으로 
하나의 라이브러리엔 이 라이브러리의 설명인 메타데이터가 자동으로 기록이 됩니다 이 기록을 기본값 1일로 설정을 하겠습니다 

4. Authentication 설정 

![4](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/9d36dfbf-4175-47e8-82bf-23366bb2c5f2)

끝으로 원격 nexus 에 접속하는 계정을 입력하고 하단 create repository 를 클릭하면 새롭게 만들어지게 됩니다 

## 로컬에서 proxy 가져다 쓰기 

![5](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/0a2c69c2-7ca0-4854-adf8-68c6705f38e0)

만들어진 리포지토리를 열어보면 처음에는 아무것도 없습니다 역시나 로컬에서 연결을 하고 요청을 주면 그때서야 필요한 파일들을 동기화 해서 가져옵니다 

## settings.xml 수정 

```

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">

<localRepository>C:\Users\kimdo\.m2\repository</localRepository>

<profiles>
        <profile>
                <repositories>
                        <repository>
                                <id>nexus</id>
                                <url>http://192.168.46.129:8081/repository/time-maven-group3--proxy/</url>
                                <releases>
                                        <enabled>true</enabled>
                                </releases>
                                <snapshots>
                                        <enabled>true</enabled>
                                </snapshots>
                        </repository>
                        
                    
                </repositories>

                <pluginRepositories>
                        <pluginRepository>
                                <id>nexus</id>
                                <url>http://192.168.46.129:8081/repository/time-maven-group3--proxy/</url>
                                <releases>
                                        <enabled>true</enabled>
                                </releases>
                                <snapshots>
                                        <enabled>true</enabled>
                                </snapshots>
                        </pluginRepository>
                </pluginRepositories>

        </profile>
</profiles>


<mirrors>
        
        <mirror>
                <!--This sends everything else to /public -->
                <id>nexus</id>
                <url>http://192.168.46.129:8081/repository/time-maven-group3--proxy/</url>
                <mirrorOf>*</mirrorOf>
        </mirror>

  
</mirrors>

<servers>
        <server>
                <id>nexus</id>
                <username>admin</username>
                <password>#########</password> <!--nexus 계정 입력 -->
        </server>
</servers>

    







</settings>


```

이렇게 아까 만들었던 nexus 를 연결하고 로컬에서 라이브러리 받기 시작하겠습니다 

```
KST: [INFO] Downloaded http://192.168.46.129:8081/repository/time-maven-group3--proxy/net/bytebuddy/byte-buddy/1.12.10/byte-buddy-1.12.10-sources.jar
23. 10. 3. 오전 8시 42분 43초 KST: [INFO] Downloaded sources for net.bytebuddy:byte-buddy:1.12.10
23. 10. 3. 오전 8시 42분 43초 KST: [INFO] Downloading http://192.168.46.129:8081/repository/time-maven-group3--proxy/net/bytebuddy/byte-buddy-agent/1.12.10/byte-buddy-agent-1.12.10-sources.jar
23. 10. 3. 오전 8시 42분 43초 KST: [INFO] Downloaded http://192.168.46.129:8081/repository/time-maven-group3--proxy/net/bytebuddy/byte-buddy-agent/1.12.10/byte-buddy-agent-1.12.10-sources.jar
```

다운로드 일부분 캡쳐해서 가져왔습니다 지금처럼 보면 아까 설정한 proxy 에서 가져오는것을 볼 수 있습니다 페이지 들어가보면 

![6](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/40c962fb-3c62-4cc4-92ec-fb29969613ed)

앞에서 만들었던 common 파일 2개가 proxy 형태로 이곳으로 넘어온것을 확인할 수 있습니다 

이렇게 우리는 nexus 가 무엇인지 부터 시작해서 간단한 사용법에서 심화 사용법까지 다루어보았습니다 




​





​

​

​

