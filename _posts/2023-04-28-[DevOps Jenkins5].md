---

title: DevOps Jenkins 처음으로 하는 jenkins 빌드2
author: kimdongy1000
date: 2023-05-01 12:43
categories: [DevOps, Jenkins]
tags: [ Jenkins ]
math: true
mermaid: true

---

우리는 지난시간에 공개되어 있는 프로젝트를 빌드하는 방법에 대해서 배웠습니다만 사실상 어딜 프로젝트를 가도 프로젝트 코드가 공개되어 있는 곳은 없습니다 
gitlab , github 프로젝트를 만들때 프로젝트 성격을 private 로 만들지 public 으로 만들지 결정하게 되는데 이때 private 로 결정하게 되면 
jenkins 는 더이상 이전같은 방식으로 프로젝트를 읽어오지 못합니다 

그럼 오늘은 비공개 되어 있는 프로젝트를 읽어오는 방식에 대해서 공부를 하겠습니다 이 비공개라는것은 내가 알고 있지 않은 프로젝트가 아닌 
내가 관리하고 있는 프로젝트 이지만 jenkins 입장에서는 인증이 필요한 프로젝트 빌드입니다 

그럼 진행을 하겠습니다

## 기존 프로젝트 private 처리 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/cf820fae-4f63-40d1-b802-9ef672d22a83)

저는 앞에서도 말했지만 gitlab 을 더욱 자주 쓰기에 gitlab 위주의 설명을 진행하겠습니다 이 사진은 해당 프로젝트의 
Settings -> General -> Visibility, project features, permissions 오시면 이 프로젝트의 공개여부를 선택할 수 있는데 지금사진처럼 private 로 신청하면 
이제 이 프로젝트로 접근을 할때 인증을 받아야 가능한 프로젝트가 됩니다 

설정을 저장을 하고 우리 프로젝트로 돌아오게 되면 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/4bd45f39-c10c-4033-96e0-a3e27da93858)

어제와는 좀 다른 붉은 글씨로 무엇인가 불안하게 적혀 있다 

```
Failed to connect to repository : Command "git ls-remote -h git@gitlab.com:kimdongy1000/jenkins_module1.git HEAD" returned status code 128:
stdout:
stderr: Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

 Command "git ls-remote -h git@gitlab.com:kimdongy1000/jenkins_module1.git HEAD 이 명령어를 통해서 git 프로젝트에 접근을 못하고 있다 
 라면서 Host key verification failed 라는 오류를 찾는다 

 자 우리가 바꾼것은 기존의 public 프로젝트를 private 프로젝트로 변경을 하게 되었다 그럼 우리도 gitbash 에서 이 프로젝트를 clone 해올때 
 gitlab 의 자격증명을 얻어서 clone 을 하게 된다 

 jenkins 도 마찬가지도 private 프로젝트를 clone 해올때 gitlab 의 자격증명을 같이 보내주어여 한다 우리는 서버에 이 자격증명을 심고 같이 보내는 방법에 대해서 
 공부를 해볼것이다 



자 그럼 여기서는 서버로 들어와야 한다 서버라고 하면 현재 jenkins 가 설치된 서버를 말하는 것이다 
![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/25c28432-a814-4fb6-ac25-3efcf488f716)

자 이 위치까지 와보자 참고로 jenkins 설치는 root 계정으로 진행을 했기에 root 계정으로 접속후 이 위치까지는 찾아와야 한다 

## ssh-keygen 만들어서 jenkins 에 등록

```

ssh-keygen -t rsa -f /var/lib/jenkins/.ssh/id_rsa

```

우리는 다음과 같은 명령으로 ssh-keygen 을 만들게 되는데 이 명령어는 유닉스 리눅스 기반에서 ssh 네트워크를 통해서 안전하게 원격 로그인 하고 파일을 전송하는 
프로토콜인데 그때의 인가를 담당하는 비대킹키 로 이루어진 암호문을 말합니다 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/519fcfad-abb2-4b18-b454-a4549eecafa5)

만드는 방법은 위의 명령어를 치고 엔터 2번 누르면된다 이때 Enter passphrase (empty for no passphrase): 그냥 엔터치고 넘어가면된다 
이는 이 암호문을 만들때 사용하는 비밀번호이다 

그렇게 만들어지면 2개의 파일이 생성이 되는데 파일 안의 내용물은 보여주지는 않을 예정이다 이 두개의 파일의 내용을 알게 되면 나의 ssh 비밀번호가 노출이 되는것이기 때문에
패턴에 대해서만 설명을 할것이다 

자 우리가 입력해야 할 택스트는 다음과 같다 id_rsa 는 jenkins 에 입력을 해야 하고 id_rsa.pub 은 gitlab 에 입력이 되어야 한다 
그럼 jenkins 어디에 입력을 하는지 살펴보자 

jenkins 로 돌아와서 좌측 jenkins 관리 -> Credentials 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/7e596794-c071-4def-9e45-deb5555a51ea) 이렇 사진이 나오는데 

하단 global 을 클릭하면 이렇게 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/20dc5b0b-a644-4cfb-9390-95f2329115c4)

이렇게 창이 하나 더 나오는데 여기서 우측에 파란색 버튼 Add Credentials 로 들어오면된다 그럼 여기에 우리가 만든 id_rsa 파을 내용물을 입력하면되는데 

먼저 kind 창을 열어서 여러개 창중에 SSH username with private key 를 선택하고 하단에 private key 를 보면 옆에 추가할 수 있는 버튼이 있는데 그것을 클릭해서 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/34422886-d8b9-4e98-9472-6c5a74e86d7d)

이렇게 입력할 수 있는 창을 만들면된다 

여기서 위에 다른 ID , Description 은 입력안해도 된다 그렇지만 저는 분간을 위해서 Description 특별한 설명을 입력을 해둘것입니다 여러분도 햐서도 되고 안하셔도 됩니다 
저는 PROD1 으로 입력할 예정입니다 

자 그럼 key 인데 다시 서버로 와서 아까 그 파일을 열어볼것입니다 그럼 이런 패턴일 것입니다 

```
-----BEGIN RSA PRIVATE KEY-----





-----END RSA PRIVATE KEY-----
```

이렇게 패턴이 되어 있을것입니다 이 내용을 앞의 --- 포함해서 남김없이 복사해서 입력을 하셔야 합니다 그리고 그 아래 Passphrase 도 입력을 안하셔도 됩니다만 
아까 이 id_rsa 를 만들때 비밀번호를 입력했다면 여기에 같은 비밀번호를 입력하셔야 합니다 
저는 그거 없이 만들었기에 넘어가겠습니다

그리고 하단에 create 를 하게 되면 하나가 만들어진게 보일것입니다 이것이 이제 jenkins 가 gitlab 에 요청할때 사용하는 인증 내용이 담겨 있습니다 
그리고 이 인가내용은 비대칭키 즉 공개키 상태이기 때문에 gitlab 는 이 내용을 디코딩할 수 있는 공개키를 주어야 합니다 그것이 바로 id_rsa_pub 입니다 이 내용을 
gitlab 에 입력을 할것입니다 

## ssh-keygen 만들어서 gitlab 에 등록 

gitlab 는 UserSettings -> SSH Key 페이지로 오시면됩니다 그럼 

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/f089a44f-dfe3-4fe0-bb5b-d34c4a4ec19d)

여기서 아까 id_rsa_pub 내용을 남김없이 모두 입력을 하면됩니다 그리고 타이틀은 아까 id_rsa 를 분간할 수 있는 무엇인가를 적어주시고 
Expiration date 의 날짜를 없애주겠습니다 이렇게 하면 만료기한이 없는 SSH key 가 만들어집니다 

그리고 하단에 addKey 를 하게 되면 준비는 끝났습니다 이제 아까 jenkins 빌드 구성하는 화면으로 돌아오면 여전히 빨간색불이 들어와 있어서 설정이 안되었다고 생각할 수 있지만 
아래 Credentials 가 보입니다 여기서 우리가 설정한 jenkins Credentials 선택해주면됩니다 

## ssh-keyscan

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/18161f4a-97e1-4373-837e-798b5ef634fe)

에러 내역이 바뀌었습니다 이제는  No ECDSA host key is known for gitlab.com and you have requested strict checking. 이렇게 되어 있는데 
gitlab.com 에 대한 host-key 를 알려달라는 것입니다 그렇기에 우리는 아까 /var/lib/jenkins/.ssh 위치에서 한가지 파일을 더 만들것입니다 

```

ssh-keyscan gitlab.com  >> /var/lib/jenkins/.ssh/known_hosts

```
이는 gitlab 에 대한 ssh-keyscan 을 나의 서버가 알고 있게금 hotst 파일에 입력을 해두는것입니다 이 명령어가 끝이나게 되면 
known_hosts 파일이 한개 더 만들어지게 됩니다 이제 끝입니다 다시 jenkins 로 돌아오게 되면

![1](https://github.com/time-kimdongy1000/ImageStore/assets/58513678/4dcaf399-e31b-46be-ac7b-7ddf08ab83fc) 

이제 그 명령어도 없어지고 깔끔한 바탕이 되었습니다 그럼 빌드를 하겠습니다 

자 그러면 이제 우리가 어제 보았던 빌드 성공 메세지가 뜨는것을 확인할 수 있습니다 오늘은 비공개로 되어 있는 프로젝트를 빌드하는 방법에 대해서 공부를 해보았습니다 
다음시간에는 jenkins 의 디렉터리 구조에 대해서 공부해볼 예정입니다

이제 gitClone 주소는 https 가 아닌 SSH 주소로 Clone 로 받아오면 됩니다