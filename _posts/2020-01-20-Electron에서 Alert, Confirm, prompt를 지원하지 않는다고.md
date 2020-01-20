---
layout: post
title: Electron에서 Alert, Confirm, Prompt 사용하면 안되는 이유? 
excerpt: ""
categories: [develop]
comments: true
image:
  feature: https://raw.githubusercontent.com/rocketlaunchr/electron-alert/HEAD/assets/electronalert.png
  credit: electron-alert
  creditlink: https://www.npmjs.com/package/electron-alert?activeTab=readme
---


### Electron 개발기
플로우는 최대한 웹의 리소스를 활용해서 설치형을 만들어야 하기에 웹뷰를 사용하여 설치형 프로그램을 개발 하고 있었다. 하지만 설치형 프로그램을 개발하시던 개발자분이 퇴사를 하고 신입 3인방중에 한명은 이 일렉트론을 맡아야했다. 어찌하다보니 일렉트론을 잡고 있는지 1년이 되었다. 그래서 어찌보면 설명하나 들어보지 못하고 코드를 보고 전에 정리를 해주신 자료를 보며 맨 땅에 헤딩을 시작했다. 


많은 커뮤니티를 통해서 일렉트론을 우리와 같이 사용하는 곳이 많은지 찾아보았지만 대부분은 웹기술을 그대로 사용해 네이티브앱을 만들어 데스크탑의 기능을 가져다 사용하는 방식을 많이 사용하고 있었고 우리와 같이 웹뷰를 위주로 사용하는 경우는 많지 않았던 것 같다. 그렇지만 지금 당장 플로우의 일렉트론을 바꿀수는 없는 노릇이다 지금 상황에서 최선을 다하고 좀 더 효율적으로 사용하는데 집중을 하기로 했다.

## 장애가 발생했다.
일렉트론 사용자들 즉 우리 플로우의 설치형 고객들이 자꾸 플로우가 먹통이 된다는 문의를 하기 시작했다. 정확한 원인 파악이 힘든 상황이 였지만 CS팀에서 의심가는 부분을 잘 지적해주셔서 원인을 찾을 수 있었다. 

## 원인은?
원인은 간단했다. 일렉트론은 Alert, Confirm, Prompt 와 같이 쓰레드를 중지시키고 데이터를 입력받는 방식을 권장하지 않고 일렉트론 함수인 dialog를 활용하도록 권장하고 있다. 하지만 우리는 일렉트론을 네이티브스럽게 사용하기보단 웹뷰를 띄우고 이를 담는 박스와 같은 역할을 하고 있었다. 그렇기 때문에 나는 웹안에서 크게 소스를 변경하지 않고 웹과 일렉트론 모두에서 사용하도록 이 문제를 해결하려고 했다.

## 해결방안
첫번째는 일렉트론 함수를 이용해서 Alert, Confirm, Prompt가 일어난 브라우져객체에 Blur() 함수를 사용해 다시 창을 누르게 하는 방식이 있었지만 이 방법은 모든 설치형을 사용중인 고객들에게 업데이트를 요구해야하는 상황이였다. 단순히 클라우드 고객만이라면 AutoUpdate 기능을 통해서 모두 업데이틀 진행하면 되지만 결국 엔터프라이즈 고객의 경우는 대부분 내부망을 사용하고 AWS와 같은 외부의 사용을 제한하는 경우가 많기에 서버 안에서 해결을 하는 편이 좋다고 생각을 했다.

그래서 내가 선택한 방법은  Alert, Confirm, Prompt을 오버라이딩하는 방법을 택했다.
결국 alert또한 window함수이다 이 함수를 오버라이딩 해서 일렉트론의 경우는 커스텀한 alert창을 만들어 보여주고 브라우저를 사용하는 고객의 경우는 그대로 alert창을 보여주면 되는 것이다.

~~~js
window.old_alert = window.alert; 

window.alert = function (message1) {
	if(getElectronYn()) {
		cmf_layerMsg("3",message1)
	} else {
        return old_alert(message1);
	}
};

// cmf_layerMsg는 우리가 커스텀하게 만든 토스트방식의 alert 함수이다.
~~~

이런식으로 코드를 짜게 되면 이미 alert을 사용중인 소스를 수정할 필요가 없어지면서 소스의 수정이 한곳에서만 이루어지기 때문에 좀더 효율적으로 시간을 아낄수 있다.

~~~js
if (confirm("로그아웃 하시겠습니까?")){
    logout()
}
~~~

confirm 또한 alert과 동일하게 기존의 confirm 함수를 저장하고 새롭게 커스텀을 진행하면 되는데 약간의 문제가 있다.
바로 confirm의 경우는 해당 시점에 데이터가 입력되는 동안 멈춰있어야하는데 커스텀을 진행하면 Click이벤트에 Yes or No의 대한 콜백함수를 정의하는데 이렇게 되면 바인딩 이벤트의 클릭을 했을때 발생하는 시점과 바로 데이터를 입력하기 전까지 멈춰있다 데이터가 입력되면 실행되는 방식의 차이로 인해 싱크를 맞추는데 문제가 생겼다.

이를 해결하기 위해 바인딩 변수에 동기적 처리를 하는방법을 찾아보았지만 찾기가 힘들었고 우선은 사용자들이 불편을 느끼고 있는 상황이라 confirm이 발생하는 부분들에 대해서 직접 모두 커스텀 처리를 진행했다. Confirm의 경우는 이미 동기 개발자가 개발해놓은 커스텀 함수를 사용해서 진행을 했다.


## 테스트

![기존의 Confirm창](https://github.com/rumor1993/rumor1993.github.io/blob/master/img/image.png?raw=true)
웹에서는 기존의 window 함수인 Confirm창이 잘나오고 있다.
![커스텀한 Confirm창](https://github.com/rumor1993/rumor1993.github.io/blob/master/img/image%20(1).png?raw=true)
Electorn에서는 새롭게 커스텀한 Confirm창이 잘나오고 있다.


## Notices
좀 더 좋은 방법이 있는 것 같은데... 일단 오류부터 해결하자
{: .notice}