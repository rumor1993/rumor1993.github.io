---
layout: post
title:  Apache 웹서버 동시접속자로 인한 503 에러가 발생했다면?
excerpt: ""
categories: [develop]
comments: true
image:
  feature: https://wisdomeweb.com/wp-content/uploads/2019/05/Tuning-Your-Apache-and-improve-performance-of-Apache-Server.jpg
  credit: wisdomeweb
  creditlink: https://wisdomeweb.com/tuning-your-apache-and-improve-performance-of-apache-server/
  styles: width:664px; 
---


### 503 service unavailable 에러가 발생했다.
엔터프라이즈 고객사에서 WAS 증설작업 이후에 팝업을 띄우는 이벤트를 실행하면 페이지를 찾을 수 없다는 오류가 발생한다고 했습니다. 우선은 팝업을 띄우기 전에 세션을 체크해서 세션이 존재하지 않으면 세션을 만들어주는 방어로직을 추가했습니다. 하지만 계속해서 오류가 발생했고 원인을 찾던 도중 503 Service Unavailable 에러가 발생했다는 소식을 듣고 스카우터를 통해서 모든 WAS서버가 살아있는지 확인을 했습니다. 이상하게도 WAS 서버에는 CPU와 GC, Service Count, TPS, XLOG 모두 이상이 없는 상태였습니다 WAS단의 원인은 아닌듯 했고 우리는 WEB서버를 확인하기 시작했다. 해당 부분을 확인해보니 에러가 발생하고 있었습니다. 에러의 내용은 너무 많은 동시접속자를 견디지 못한다는 오류 였습니다. 

## 원인파악
원인은 서버를 증설하면서 아파치 설정부분도 변경이 이루어 졌는데 해당 부분에서 고객사의 인프라팀이 동접자수 관련설정이 낮게 설정 된 것으로 나타났습니다. 아파치의 로그를 확인해보면 server reached MaxRequestWorkers setting, consider raising the MaxRequestWorkers setting 서버의 설정한 MaxRequestWorkers 수치에 도달했으니 MaxRequestWorkers 값을 수정을 요구한다고 하네요. 그렇다면 MaxRequestWorkers가 무엇일까? 이 부분에 대해서는 Apache에 MPM(Multi-Processing Module)를 알아야합니다.

## Apache에 MPM(Multi-Processing Module)
아파치 웹서버는 멀티 프로세스 방식으로 요청(Request)을 처리하는데 이때 prefork와 worker의 방식을 선택해서 처리를 합니다.  

# Prefork 방식
Prefork 방식은 자식프로세스가 싱글쓰레드로 동작한다고 합니다. 클라이언트로 요청을 받게 되면 하나의 프로세스가 이를 처리하는 방식을 말합니다 한 자식 프로세스당 하나의 쓰레드를 사용하는 방식을 PreFork라고 하는데 이 방식은 안정적인편이라서 아파치의 기본설정값으로 설정이 되어집니다. 아무래도 요청당 하나의 프로세스가 이를 처리하기 때문에 안정성이 높아 집니다 하지만 PreFork 방식은 Worker 방식보다 자원을 많이 사용하는 단점이 있습니다 즉, 동시접속자 많아진다면 Worker 방식에 비해 많은 자원을 사용하게 됩니다

<!-- https://img1.daumcdn.net/thumb/R720x0.q80/?scode=mtistory2&fname=http%3A%2F%2Fcfile24.uf.tistory.com%2Fimage%2F225E6A40579848BB2B1A26 -->

# worker 방식
Worker 방식은 자식 프로세스가 멀티 쓰레드로 동작한다고 합니다. 클라이언트로 요청을 받게 되면 프로세스가 이를 처리하는 PreFork와 달리 하나의 쓰레드가 처리하는 방식 입니다. 한 자식의 프로세스당 여러 개의 쓰레드를 가지고 있기 때문에 Worker 방식이 Prefork보다 메모리를 적게 사용하는 장점을 가지게 됩니다. 
그래서 동시접속자를 처리하는데 유리합니다. 또한 스레드간에 메모리를 공유하고 있다고 하네요

<!-- https://img1.daumcdn.net/thumb/R720x0.q80/?scode=mtistory2&fname=http%3A%2F%2Fcfile8.uf.tistory.com%2Fimage%2F2772D740579848BC180D97 -->


~~~
// PreFork 방식
# /usr/local/apache/conf/extra/httpd-mpm.conf
<IfModule mpm_prefork_module>
    StartServers 5
    MinSpareServers 5
    MaxSpareServers 10
    MaxRequestWorkers 300
    ServerLimit 300
    MaxConnectionsPerChild 0
</IfModule>
~~~

- startServer : 아파치서버의 자식프로새스 개수를 지정합니다
- MinSpareServers : 서버 프로세스의 최소 수, MinSpareServers가 유휴 상태보다 적은 경우 부모 프로세스는 새 자식 프로세스를 생성합니다
- MaxSpareServers : 서버 프로세스의 최대 수, MaxSpareServers가 유휴 상태보다 큰 경우 부모 프로세스가 초과되는 프로세스를 중지 시킵니다.
- MaxRequestWorkers, ServerLimit : 기본값이 256 이기 때문에 MaxRequestWorkers 값이 256보다 작으면 따로 적을 필요가 없으며, 256보다 크면 그와 같은 값으로 설정해야합니다.
- MaxRequestPerChild : 클라이언트들의 요청 개수를 제한합니다. 자식 프로세스가 이 값 만큼의 클라이언트 요청을 받았다면 이 자식 프로세스는 자동으로 Kill이 됩니다 (0은 무한대)

~~~
// Worker 방식
# /usr/local/apache/conf/extra/httpd-mpm.conf
<IfModule mpm_prefork_module>
    StartServers 3
    MaxClients 150
    MinSpareThreads 75
    MaxSpareThreads 250
    ThreadsPerChild 25
    MaxRequestWorkers 400
    MaxConnectionsPerChild 0
</IfModule>
~~~

- StartServers(Default 3) : 시작시에 생성되는 서버 프로세스의 개수, 자식 프로세스의 수는 부하에 따라 동적으로 변경되기 때문에 이 값은 큰 의미가 없습니다.
- ServerLimit (default : 16) : 구성 가능한 child 프로세스의 제한 수. ServerLimit 값이 필요 이상 높게 설정 된다면, 불필요한 공유 메모리가 할당 되므로 적절한 설정 필요합니다. MaxClient 와 ThreadPerChild 에서 요구한 프로세스 수보다 높게 설정하지 마시기 바랍니다.
- MaxClient (default : ServerLimit * ThreadsPerChild) : 동시에 처리될 최대 커넥션(request)의 수, MaxClients 수치를 초과한 후 온 요청들은 ListenBackLog에 의해 대기상태가 됩니다. ThreadsPerChild 옵션과 매우 긴밀하게 작용, 동시접속자가 많을 경우, MaxClient값을 증가시켜야 합니다. OS의 FD(File Descriptor)값을 증가 시켜 MaxClient 의 상한값을 증가시키시기 바랍니다.
MaxRequestWorkers `was called MaxClients before version 2.3.13. The old name is still supported`. (이전버전의 경우 maxClients)
- MinSpareThreads(default 75) : 최소 thread 개수, 서버에 idle 쓰레드가 충분하지 않다면 child 프로세스는 idle 쓰레드가 MinSpareThreads 보다 커질때까지 생성됩니다.
- MaxSpareThreads(default 250) : 최대 thread개수, 서버에 너무 많은 idle 쓰레드가 존재하면 child 프로세스는 idle 쓰레드가 MaxSpareThreads 수보다 작아질 때까지 kill 됩니다.
- ThreadPerChild : 개별 자식 프로세스가 지속적으로 가질 수 있는 Thread의 개수.
- MaxRequestPerChild : 자식 프로세스가 서비스할 수 있는 최대 요청 개수
- ThreadLimit (default : 64) : child 프로세스의 라이프주기 동안 ThreadsPerChild 의 최대 설정값을 설정합니다. ThreadLimit 가 ThreadsPerChild 보다 훨씬 높게 설정된다면, 여분의 미사용 공유 메모리가 할당될 것입니다. ThreadLimit 과 ThreadsPerChild 모두 시스템이 감당할 수 있는 값 보다 높게 설정하면, 아파치가 기동되지 않거나 시스템이 불안정하게 될 수 있습니다.최대 예상 ThreadsPerChild의 설정보다 높게 설정하면 안됩니다.

https://tkdev.tistory.com/11
동시접속자 메모리를 계산하는 부분

## 조치사항
~~~
// Worker 방식
# /usr/local/apache/conf/extra/httpd-mpm.conf
<IfModule mpm_prefork_module>
    StartServers 5
    ServerLimit 256
    MinSpareThreads 75
    MaxSpareThreads 250
    ThreadsPerChild 64
    MaxRequestWorkers 8192
    MaxConnectionsPerChild 0
</IfModule>
~~~

아파치 에러 로그에 나온 server reached MaxRequestWorkers setting, consider raising the MaxRequestWorkers 내용처럼 `MaxRequestWorkers`의 설정값을 높게 변경하도록 했습니다. 즉, ServiceLimit 256 이고 ThreadsPerChild 64로 설정했기 때문에 총 16,384‬개의 동시처리가 가능해집니다. 하지만 실제로 가동되는 프로세스의 수를 계산해야하는데 MaxRequestWorkers를 8192 /  ThreadsPerChild 64 나눈 128개가 가동되는 상태가 됩니다. 최초의 5개의 프로세스로 시작을 해서 한개의 프로세스가 64개 요청처리가 가능한 상태에인데 한개의 프로세스가 추가 될때마다 64개의 요청이 더 처리가 가능해집니다. 그렇다면 해당 설정값은 3,200개의 요청을 처리가 가능해집니다.
 
## Notices
아파치의 MPM중 Event에 대해서도 알아보아야겠다. WAS에 대해서도 좀더 많은 지식을 갖도록 노력해야겠다. 
{: .notice}

