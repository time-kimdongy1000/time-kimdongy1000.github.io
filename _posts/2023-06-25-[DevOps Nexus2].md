---

title: DevOps Nexus 2 Maven Centeral
author: kimdongy1000
date: 2023-06-25 09:23
categories: [DevOps, Nexus]
tags: [ Nexus ]
math: true
mermaid: true

---

지난시간에 우리는 Nexus 가 무엇인지 그리고 설치하는 방법에 대해서 알아보았습니다 오늘은 Maven Centeral 에 대해서 알아보도록 하겠습니다 

Maven Centeral 말그대로 중앙연결소입니다 우리가 아는 중앙연결소는 https://mvnrepository.com/ 이 부분을 말하며 실제 nexus 를 설치하면 기본적으로 이 주소를 사용하고 
바로 연결까지 해놓은 Repository 가 있습니다 

## maven - central Repository

![maven central](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/76250875-36b6-4849-a66e-f3fdf8de80bc)

바로 이부분입니다 이 부분의 세팅은 

![maven central](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f6072da5-d469-4e5a-a00b-9b0eea2b3fdb)

이렇게 상단에 톱니바퀴 누르고 왼쪽 메뉴에서 Repositories 를 클릭하면됩니다 그리고 maven - central 를 클릭하면 이 중앙저장소의 기본적인 설정을 볼 수 있습니다 


![maven central1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/76fa0fae-e6d8-4413-805a-bcb6a7408904)

그럼 이헌 화면이 나오게 되는데 내부설정은 제가 다음 설정에서 쓰는 나만의 Repository 를 만드는것으로 확인을 하겠습니다 
그리고 이 repositorty 가 중앙저장소랑 연결되어 있다는 것을 알 수 있는것은 하단 Proxy 에 Remote storage 이렇게 되어 있습니다 이 주소를 인터넷에 입력을 하면 

![maven central](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/e3bdb0da-dda1-4c1e-9528-04241e338192)

중앙으로 연결된 모든 maven 아카이브가 저장되어 있는 사이트로 연결이 됩니다 이 주소를 기반으로 라이브러리를 다운로드 받고 업데이틀 하게 됩니다 


## 로컬에 연결하기 

그럼 이 nexus 를 어떻게 나의 로컬하고 연결을 할 수 있을까? 잠깐 현재 상황을 간단한 그림으로 알아보자 

![maven central](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/966c04ea-ee73-4e14-84eb-6263304f8b93)

이게 기본으로 개발자가 Nexus 없이 사용할떄의 환경입니다 라이브러리가 필요하면 곧바로 중앙처리소에서 다운로드를 진행하게 됩니다 


![maven central1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/43e03f1c-5765-4dd1-afd7-70a06e10057d)

이제 이게 우리의 환경입니다 우리는 중앙처리소에서 받는게 아니라 우리가 이 라이브러리 필요한것을 nexus 에 요청하면 neuxs 가 중앙처리소에 요청해서 
적절한 라이브러리를 자신도 받고 사용자에게 내려줍니다 

그러면 이게 단계가 하나 더 늘어난것이니 불편한게 아니냐 할 수 있지만 실제로 단계가 하나 더 늘어났으니 불편한게 맞습니다 단순 라이브러리 다운로드로만 활용하게 된다면 
하지만 nexus 는 기본적으로 인터넷이 안되는 환경에서 사용할 수 있는 서드파티 애플리케이션임을 망각하시면 안됩니다 이 부분이 nexus 를 사용하는 핵심입니다 

이제 우리는 로컬환경에 이 nexus 를 연결해서 라이브러리를 다운로드 받겠금 만들어보겠습니다 


## settings.xml 

아마 한번씩을 들어보았을것이다 기본적으로 개발환경을 설치하면 제일 앞에서 어디서 개발환경을 읽을것인지 지정을 할 수 있습니다 이 부분을 수정을 하겠습니다 

![settings xml ](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/02f68dbd-6aba-43a4-8bca-4d830cf4136b)

저는 인텔리j 를 사용해서 개발을 진행함으로 인텔리j 위주로 설명을 드리겠습니다 (settings.xml 위치는 이클립스가 사용하는 위치와 동일합니다)


```

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">

<localRepository>C:\Users\kimdo\.m2\repository</localRepository>
<mirrors>
        <mirror>
                <!--This sends everything else to /public -->
                <id>nexus</id>
                <mirrorOf>*</mirrorOf>
                <url>http://192.168.46.131:8081/repository/maven-central/</url>
        </mirror>
</mirrors>


<servers>
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>#####</password>
    </server>
</servers>




<profiles>
        <profile>
                <repositories>
                        <repository>
                                <id>nexus</id>
                                <url>http://192.168.46.131:8081/repository/maven-central/</url>
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
                                <url>http://192.168.46.131:8081/repository/maven-central/</url>
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

</settings>



```

이 파일을 열어보면 이런식으로 어떤 설정을 할 수 있는 xml 파일이 있습니다 여기에 우리는 우리가 설치한 nexus 의 central 서버를 연결을 하겠습니다 

```
<mirrors>
        <mirror>
                <!--This sends everything else to /public -->
                <id>nexus</id>
                <mirrorOf>*</mirrorOf>
                <url>http://192.168.46.131:8081/repository/maven-central/</url>
        </mirror>
</mirrors>

```
부분의 url 부분에 주소를 입력합니다 이 주소는 

![maven central](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/a1a0331c-4242-4ee8-b5e6-90df370a59e4)

repository 의 url 을 클릭하면 해당 repository 의 주소가 나오게 됩니다 이 주소를 입력한 다음 저장을 하고 

![settings xml2](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7a900953-4267-41a6-a03f-7f36b867ea3c)

이렇게 옆에 인텔리j 의 부분의 오버라이드 부분을 체크를 해줍니다 그러면 이제 우리는 기본적인 연결은 완료되었습니다 
그러면 다운로드 해보겠습니다


그리고 비밀번호는 가렸습니다 그리고 다시 인텔리 j 로 돌아와서 기존 라이브러리를 삭제하고 다시 다운로드 하면 이제 nexus 를 통해서 다운로드 진행하게 됩니다 

![central ](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/b2c2a0dc-2858-4a94-896c-7bf72036a664)

그렇게 되면 이제 이제 아까 그 주소에서 중앙저장소와 연결된 부분과 동기화가 시작이 되는것입니다 

즉 사용자가 요청을 하기 전까지 nexus 는 중앙저장소와 동기화 하지 않습니다 사용자가 필요로 하는 라이브러리를 요청하면 그떄서야  하는 작업을 진행하게 됩니다 
이렇게 되면 이제 로컬하고 저의 nexus 하고 연결은 완료된것입니다 

이렇게 해서 우리는 nexus 와 로컬환경을 연결하는 방법에 대해서 공부 해보았습니다 
