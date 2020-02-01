---
layout: post
title:  Electron 빌드를 Jeknins와 Git을 이용해서 해보자
excerpt: ""
categories: [develop]
comments: true
image:
  feature: electron+jenkins.png
  credit: rumor1993
  creditlink: https://rumor1993.github.io/
  styles: width:664px; 
---

### Electron Build를 굳이 Jenkins를 사용하는 이유?
* 첫번째이유  
플로우에서는 클라우드용 설치파일과 인하우스용 설치파일이 존재합니다. 그렇다보니 해당 인하우스 고객의 설치파일이 필요할때마다 이를 직접 electron-builder를 이용해서 설치파일을 만들어 전달을 해주고 있습니다. 인하우스 고객의 수가 많아 질수록 빌드를 진행해야할 빈도수가 늘어나게 되고 이를 다시 기획팀에 전달해주는 작업이 생각보다 많은 시간을 사용하게 되고 있어 이를 Jenkins와 Git을 이용해서 좀 더 효율적인 방법으로 만들기 위해 Jenkins와 Git을 사용하게 되었습니다. Jenkins에 만들어놓은 프로젝트에서 빌드버튼만 누르면 설치형 파일이 만들어지고 이를 기획팀에서는 개발팀을 거치지 않고 파일을 가져가서 해당 고객사에 전달이 가능하도록 변경할 예정입니다. 

* 두번째 이유  
Mutli Pltform Build을 지원하기 위해서 입니다. 아무래도 Windows에서는 Mac용 설치형 파일을 빌드하지 못하는 상황입니다. 하지만 Mac에서는 Windows와 Mac용 설치파일 두가지를 모두 빌드를 할 수 있습니다. (단, 카탈리나버전의 경우는 win32버전은 빌드가 안된다고 하네요)   


또한 Git을 이용해서 코드의 수정이 이루어지면 이를 WepHook을 이용해서 최신버전을 자동으로 빌드 및 버전관리 하도록 진행합니다. 

## Jenkins를 설치해보자
Windows, Mac, Linux에 모두 Jenkins를 설치해보고 테스트를 진행해봤지만 Mutli Pltform Build를 위해서 회사에 남아있는 맥킨토시를 이용하기로 했습니다.

Mac에서 Jenkins 설치하기
* /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

우선 맥에서 손쉽게 패키지 관리를 진행해주는 HomeBrew를 설치 해주세요   

* $ brew install jenkins 

 젠킨스를 설치하기 위해서는 Java가 설치 되어야  합니다 이미 존재한다면 상관 없지만 없다면 설치를 해주세요

 * $  brew services start jenkins 
 
 해당 명령어를 통해서 Jenkins를 실행해주면 됩니다. 실행이 제대로 되었는지 확인하기 위해서는 로컬컴퓨터라면  http://localhost:8080 로 접속해보면 아래와 같은 화면이 나타납니다.

 
![젠킨스 설정화면](http://theeye.pe.kr/wp-content/uploads/2016/10/jenkins-installation-with-homebrew01-600x529.png){: style="margin:0px"}

*  $ cat /Users/dennis/.jenkins/secrets/initialAdminPassword

명령어를 사용하면 설정창에 입력해야 할 패스워드 값이 나타나게 되는데 그 값을 입력해주시면 됩니다.


![젠킨스 설정화면](http://theeye.pe.kr/wp-content/uploads/2016/10/jenkins-installation-with-homebrew02-600x531.png){: style="margin:0px"}

우리는 일반적인 플러그인을 먼저 설치하면 되는상황이니까 왼쪽을 선택해주시면 됩니다. 추후에 추가적인 플러그인 설치가 가능하니까 크게 문제가 없습니다.

만약 설치중에 오류가 발생한다면 네트워크적인 문제가 발생한 경우이거나 방화벽설정으로 인해서 다운로드에 제약이 생긴경우이니 해당부분을 확인해보시면 될 것 같습니다

![젠킨스 설정화면](http://theeye.pe.kr/wp-content/uploads/2016/10/jenkins-installation-with-homebrew04-600x416.png){: style="margin:0px"}

플러그인 설치이후에 관리자계정을 만드는 창이 나타나는데 필요한 정보들을 입력해주시면 됩니다.

이렇게 되면 젠킨스 설치 및 기본설정이 끝이납니다. 하지만 우리는 Git과 Jenkins를 연동해야하고 localhost뿐만 아니라 외부에서도 접속이 되도록 추가적인 설정을 진행해야 합니다.

## Git (Private Repository) + Jenkins 연동하기
우선 Private Repository를 Jenkins에서 접속하기 위해서는 SSH (Secure Shell Protocol) 방식으로 연동을 진행해야 합니다.

* $ cd ~/.ssh
* $ ls

해당  폴더로 이동한 후 ssh키가 존재하는지 확인합니다. (폴더가 없다면 mkdir ~/.ssh)로 폴더를 만들어주세요

* ssh-keyget -t rsa -f ~/.ssh/"KeyName"

`KeyName` 부분에 ssh키 이름으로 설정할 값을 입력하시면 됩니다. 비밀번호를 입력하는 부분이 나타나는데 저는 설정하지 않기 위해서 바로 Enter를 눌렀습니다. 다 생성이 완료 되었다면 ls -al 로 비밀키/공개키 생성여부를 확인합니다.

`KeyName` , `KeyName`.pub 파일이 생겨있을텐데요 그렇다면 성공적으로 키를 생성한 것 입니다 이제 GitHub에서 설정을 하겠습니다.

![깃허브 설정화면](/img/gitgub_ssh.png){: style="margin:0px"}

위에 보이는 이미지처럼 프로필을 클릭하면 나오는 settings 부분을 클릭하고 SSH and GPG keys 부분을 클릭하시면 SSH Key를 생성하는 페이지가 나타납니다 여기서 New SSH Key를 클릭하시면 됩니다.

![깃허브 설정화면](/img/git_newSSH.png){: style="margin:0px"}

방금전에 생성한 공개키값을 Key 부분에 입력해주시면 됩니다 Title 부분은 원하시는대로 설정하시면 됩니다.  

* cat ~/.ssh/"KeyName".pub (공개키)

ssh-rsa 부분부터 복사해서 넣어주시면 됩니다. 그리고 add SSH Key를 눌러주시면 SSH키가 생성이 완료 되었습니다. 이제 다시 Jenkins로 가서 설정을 변경하도록 하겠습니다.

Credentials -> System -> Global credentials -> Add Credentials
* cat ~/.ssh/"KeyName" (비밀키)

Kind 부분은 SSH로 변경해주시고 UserName에는 Jenkins Job에서 보여줄 인증키 이름 입니다 Private Key는 비밀키를 입력해주시면 됩니다

![Jenkins](https://t1.daumcdn.net/cfile/tistory/99B2374E5D57601A32){: style="margin:0px"}
![Jenkins](/img/jenkins.png){: style="margin:0px"}
![jenkins](/img/jenkins2.png){: style="margin:0px"}

소스 코드 관리에서 Git을 선택해주시고 Repository URL을 넣어주시면 됩니다. Credentials 부분에는 아까 만든 젠킨스 인증 UserName을 선택해주시면 됩니다.


여기서 깃주소를 입력할때 실수하시는 부분이 깃주소 그대로를 입력하면 안되고 ssh 주소를 입력 해주셔야 합니다.
![ssh](/img/sshues.png){: style="margin:0px"}
![ssh](/img/sshues2.png){: style="margin:0px"}


## Notices
2부에서는 locahost환경에서 외부에서 접속하도록 설정하는 방법에 대해서 포스팅하도록  하겠습니다. (:
{: .notice}