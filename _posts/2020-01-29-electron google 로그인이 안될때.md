---
layout: post
title:  Electron Google Oauth "계정을 안전하게 유지할 수없는 브라우저 나 앱에서 로그인하려고합니다." 에러가 발생했다!
excerpt: ""
categories: [develop]
comments: true
image:
  feature: https://pragli.com/blog/content/images/2019/12/desktop-authentication-warning-message.png
  credit: google
  creditlink: https://pragli.com/blog/how-to-authenticate-with-google-in-electron/
  styles: width:664px; 
---

### 일렉트론 Google 로그인이 안된다?

설치형 데스크탑 버전에서 갑자기 구글 `Oauth` 통한 로그인을 하려고 하면 아래와 같은 오류가 발생하게 되었습니다. 원인을 확인해보니 구글에서는 이전부터 비표준 브라우저에 대해서 `Oauth` 인증을 허용하지 않는다고 발표했는데 이 부분이 Electron 즉, Chromium 기반의 Electron 또한 대상에 대상에 포함이 된 부분인 것 같습니다 ㅠㅠ

![자바스크립트 동작원리](https://pragli.com/blog/content/images/2019/12/desktop-authentication-warning-message.png)

## 다른 플랫폼은?
우선 해결방안을 찾기 위해 기존의 일렉트론을 사용하고 `Oauth`를 사용하는 플랫폼들을 직접 설치해보고 실험을 해보았는데 기존의 일렉트론 내에서 진행하던 로그인 방식을 웹뷰를 띄우고 해당 값을 웹에서 받고 그 값을 다시 일렉트론으로 보내주는 방식을 택하고 있는 것으로 보였습니다.

## 생각보다 간단한 해결방안
플로우도 웹으로 로그인을하고 얻은 값을 다시 일렉트론으로 쏴주어야 하나 고민을 했지만 구글의 방대한 자료와 번역을 이용해 많은 서치를 하던 도중 생각보다 간단한 방법을 찾게 되었다. 비 표준 브라우저에서 `Oauth`에 대한 인증을 허용하지 않는다는 것은 결국 표준 브라우저에서는 이를 허용하고 있다는 뜻이다. 그래서 구글 인증을 하는 BrowserWindow에 옵션중에 loadUrl 함수 부분에 Option 값으로 `userAgent`를 설정하는 부분이 있는데 이부분에 단순하게 userAgent : chrome 이라는 값을 넣어주면 문제가 해결된다. 

~~~js
mainWindow.loadURL("https://google.co.kr",{userAgent: 'Chrome'});
~~~

![Oauth 동작원리](https://s3-ap-northeast-2.amazonaws.com/opentutorials-user-file/module/2398/6264.png)

이번 기회에 생활코딩의 Oauth에 대한 강의도 듣고 이를 어떻게 해결할지 고민했지만 생각보다 해결책이 간단해서 허무했지만 결국 설치형 버전을 다시 배포해야하고 고객들을 강제 업데이트를 시켜야하는 작업이 생겼다.


## Notices
구글은 역시.. 대다나..
{: .notice}

