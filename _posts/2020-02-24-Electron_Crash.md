---
layout: post
title:  Electron 앱이 죽는다고!?
excerpt: ""
categories: [develop]
comments: true
image:
  feature: https://lab.getapp.com/wp-content/uploads/2018/07/HEAD-Write_killer_bug_reports_with_these_5_bug_tracking_apps_Hero.png
  credit: getapp
  creditlink: https://lab.getapp.com/5-bug-tracking-apps-for-killer-bug-reporting/
  styles: width:664px; 
---

### Electron 앱이 죽는다는데?

업무를 보던 도중 CS팀에서 일렉트론을 기반으로 만든 플로우의 설치형 프로그램을 사용하는 고객중에 파일을 첨부하면 앱이 죽어버린다는 내용의 에러가 들어왔습니다. 동일한 증상을 확인하기 위해 앞에 놓인 데스크탑과 맥을 통해서 동일하게 테스트를 진행했지만 일렉트론앱은 죽지 않았고 고객분에게 연락을 하여 정확한 원인을 파악하기로 했습니다. 고객이 올린 파일을 전달받고 동일한 경로를 만들고 파일을 첨부해보고 반복적으로 실행을 해도 동일한 증상이 나타나지 않았습니다. 그렇다면 이 문제를 어떻게 해결해야 하나 고민을 하다가 모바일 팀에서 앱이 죽으면 CrashReport 를 통해서 원인을 파악하는 것이 생각이 났습니다.


## Electron CrashReport 

다행스럽게도 일렉트론은 CrashReport를 지원하고 있었습니다 Electron 공식 문서를 가보니 CrashReport를 설정하는 부분이 자세하게 설명이 되어져 있었습니다. 하지만 대부분의 설명은 유료 기반의 플랫폼과 연동을 하는 방식에 대해서 설명이 되어져 있었습니다.

![BackTarce](https://backtrace.io/wp-content/uploads/2017/11/FREE-FOR-ELECTRON-MAINTAINERS-compressor.png){: style="margin:0px"}

BackTrace의 경우는 유료이다보니 한달의 무료기간을 사용이 가능하지만 고객에게 발생하는 에러가 한달뒤에 나타날지 한달안에 나타날지 정확한 기간을 알지 못하는 시점에서 해당 크래쉬 리포트를 적용한 버전의 설치형을 전달하는 것보단 플로우의 개발서버에 Crash 정보를 보낼 수 있도록 서버를 띄우고 Crash의 정보를 얻을 수 있도록 만들기로 했습니다.


## Node Express로 Crash Server 만들기

![Server](https://encrypted-tbn0.gstatic.com/images?q=tbn%3AANd9GcRu-_JOJpNPBORIRqzgifz6RsD0QvC3eNz7dAUsn0UgsmFESB0s){: style="margin:0px"}

간단하고 빠른시간안에 Crash Server를 띄우려고 했기 때문에 Node Express를 이용해서 서버를 활용 했습니다.

우선은 클라이언트단에서 에러가 발생해 앱이 죽는 상황이 발생할때 에러를 서버로 보낼 수 있도록 일렉트론 앱에 아래와 같이 CrashReporter를 사용하기 위해 변수와 함수를 선언해주세요
``` js
const { app, BrowserWindow, crashReporter } = require('electron');

crashReporter.start({
  productName: 'TaskApp',
  companyName: 'Thorsten Hans',
  submitURL: 'http://"IP주소 OR 도메인주소:3000"/api/app-crashes',
  uploadToServer: true
});

```
클라이언트 단에서는 단순하게 해당 부분만 적어주면 처리가 끝납니다. 간단하죠?
자 이제 중요한 서버를 구성해보도록 하겠습니다.

아래와 같이 express를 통해서 3000포트를 열어주고 코드를 작성해주세요
그러면 electron에서 Crash가 발생했을때 `http://"IP주소 OR 도메인주소:3000"/api/app-crashes` 주소로 Post형식으로 에러가 발송되고 아래와 같은 로직을 타게 됩니다.

```js
const express = require('express'),
  multer = require('multer'),
  bodyParser = require('body-parser'),
  path = require('path'),
  fs = require('fs'),
  http = require('http'),
  miniDumpsPath = path.join(__dirname, 'app-crashes');

const app = express(),
  server = http.createServer(app);

let upload = multer({
  dest: miniDumpsPath
}).single('upload_file_minidump');

app.post('/api/app-crash', upload, (req, res) => {
  req.body.filename = req.file.filename
  const crashData = JSON.stringify(req.body);
  fs.writeFile(req.file.path + '.json', crashData, (e) => {
    if (e){
      return console.error('Cant write: ' +  e.message);
    }
    console.info('crash written to file:\n\t' + crashData);
  })
  res.end();
});

server.listen(3000, () => {
  console.log('running on port 3000');
});

```

이제 실제로 일렉트론을 사용하다가 Crash가 발생했을때 서버에 에러 덤프파일이 생기는지 테스트 하기 위해서 클라이언트단에 Crash 관련 소스를 적어 넣은 부분 아래에 `process.crash()` 함수를 `settimeout` 함수로 5초뒤에 실행을 시켜서 앱을 죽여보았는데 성공적으로 서버에 덤프파일이 생성된 부분을 확인했습니다.

그렇게 새로 ELectron 앱을 빌드해서 해당 고객분께 전달을 해드렸고 이제 에러가 발생하면 날아오는 Crash 정보를 통해서 원인을 파악하고 이를 수정하면 됩니다!


![bug](https://cdn.clien.net/web/api/file/F01/8038903/12e55f8f96b042.png){: style="margin:0px"}


## Notices
 얼른 오류를 잡고싶다.... ㅎㅎ
{: .notice}