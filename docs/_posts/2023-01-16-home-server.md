---
title: 집에있는 컴퓨터로 웹 서비스 배포 준비하기
published: false
tags: network, nginx
---

향후 완성될(언제가 될지 모르는) 토이 프로젝트 웹 서비스 배포를 준비하기 위해 집에있는 컴퓨터를 서버로 사용해 보겠습니다.
예제에서는 매우 간단한 프론트와 백엔드 데모 프로그램을 배포하겠습니다. 그리고 포스팅을 위해 일회성으로 도메인도 구입하여 연결 해보겠습니다.

---

# 1. 서버 준비

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-01.jpeg)

## 1.1 우분투 설치

저를 포함한 대부분의 사람들은 집에 서버나 워크스테이션급의 컴퓨터가 없습니다. 그러니 집에 굴러다니고 있는 라즈베리 파이를 하나 골라서 서버로 사용해 볼까 합니다.
저는 라즈베리 파이에 우분투 서버 20을 설치하여 사용중입니다. 라즈베리 파이에 우분투를 설치하는 방법은 [여기](https://ubuntu.com/download/raspberry-pi)에서 확인하실 수 있습니다.

> 라즈베리 파이의 전원 공급은 전용 어댑터를 사용하거나 5V 3A 출력의 어댑터를 사용해야 합니다.
> 
> 또한 어댑터를 멀티탭에 연결하는 것 보다, 벽면 콘센트에 바로 연결하거나 전기를 안정적으로 공급할 수 있는 용량이 큰 멀티탭을 사용하는것이 좋습니다.
> 전원공급이 불안정하면 부팅이 안되거나 비정상적인 재시작이 발생했습니다.

라즈베리 파이가 없더라도 서버로 사용할 항상 켜져있는 컴퓨터가 있다면 상관없습니다.

## 1.2 서브 네트워크에 연결

서버가 준비되었다면 우리집의 서브 네트워크에 연결합니다. 랜선을 사용하여 유선으로 연결할 수 도 있고, 와이파이로 무선 연결을 해도 좋겠지요.

> 저는 우분투 설치과정에서 와이파이 연결 설정을 했었지만 잘 동작하지 않았습니다. 검색 결과 해결한 방법은 아래와 같습니다.
> 
> `access-points` 아래 뎁스의 따옴표에 와이파이 이름을 입력합니다. 그리고 그 다음 뎁스의 `password` 항목에 와이파이 비밀번호를 입력합니다.
> 두 경우 모두 따옴표로 감싸야합니다. 또한 파일포맷이 `YAML` 이므로, 들여쓰기를 반드시 지켜줘야 합니다.
> ![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-02.png)

저는 아이피타임 공유기를 사용중입니다. 공유기가 DHCP 서버 역할을 하므로, 방금 연결된 기기에 알맞은 서브넷 IP 를 할당했을 겁니다.
아이피타임은 브라우저로 접속할 수 있는 GUI 설정을 지원합니다. 웹 브라우저를 열고 'http://192.168.0.1' 으로 접속하여 관리자 계정으로
로그인 합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-03.png)

메뉴 '고급 설정 -> 네트워크 관리 -> 내부 네트워크 설정' 에서 방금 연결한 파이의 아이피를 확인 할 수 있습니다.
저는 192.168.0.8 를 할당 받았습니다. 아이피 옆에 있는 데이터는 MAC 주소입니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-04.png)

혹은 위와 같이 `ifconfig` 명령을 통해 파이 터미널에서 아이피를 직접 확인하실 수 도 있습니다.

## 1.3 SSH 연결

만약 라즈베리 파이에 모니터가 연결되어 있어서 항상 파이의 터미널을 사용할 수 있다면 ssh(secure shell) 사용은 필수는 아닙니다.
저는 파이는 전원과 와이파이만 연결되어있고 별도의 모니터는 연결되어 있지 않으므로, ssh 접속을 사용하겠습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-05.png)

먼저 파이에서 `systemctl status ssh` 명령을 사용하여 ssh 데몬이 정상적으로 실행 중인지 확인합니다. 대부분은 기본적으로 부팅시
ssh 서비스가 실행되지만 어떤 배포판에서는 기본적으로 비활성화 되어 있는 경우를 봤습니다.
만약 ssh 서비스가 inactive 라면 `systemctl restart ssh` 명령으로 실행시켜줍니다.

이제 라즈베리 파이 외부의 원격지에서 ssh 로 접속하겠습니다.
`ssh <your_account>@192.168.0.8` 명령으로 접속합니다. <your_account> 부분을 여러분의 우분투 계정이름으로 바꿔주세요.
마찬가지로 아이피도 여러분의 서버의 서브넷 아이피로 바꿔주십시오. 접속시 해당 계정의 비밀번호를 입력합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-06.png)

ssh 로 접속하였습니다. 마지막 줄의 from 192.168.0.5 는 저의 데스크탑 컴퓨터의 서브넷 아이피입니다.
다음으로 매번 'ssh 계정@호스트' 를 타이핑 하기 귀찬으니 ssh 설정파일에 호스트를 등록하겠습니다.
~/.ssh/ 로 이동해서 'config' 파일이 있다면 수정하고 없다면 생성해줍니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-07.png)

```shell
Host <호스트 별칭>
  HostName <아이피 혹은 도메인>
  User <계정명>
  IdentityFile <ssh 키파일 경로>
```

위의 양식대로 입력해줍니다('<>' 기호는 제거해주세요). 그 다음엔 `ssh <호스트 별칭>` 으로 접속이 가능합니다. 키를 저장하지 않았다면 계정의 비밀번호를 입력 후 접속할 수 있습니다.

# 2. 정적 페이지 배포

HTTP 서버를 사용해서 매우 간단한 정적 HTML 페이지를 응답하는 서버를 설정하겠습니다. HTTP 서버는 nginx 를 사용합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-08.png)

테스트용 이므로 아주 간단한 HTML 문서를 작성했습니다. 파이에서 바로 작성해도 되고 다른 컴퓨터에서 작성 후 FTP, 이동식 미디어장치 혹은 scp 명령 등으로
파일을 서버에 복사해줍니다.

> scp(secure copy) 사용법은 아래와 같습니다. 위에서 ssh 호스트 설정을 하셨다면 그 호스트별칭을 사용할 수 있습니다.
> 원격지의 복사 경로는 절대경로를 사용해주시고, 홈을 의미하는 '~' 는 사용하지 말아주세요.
>
> `scp ./file your_account@192.168.0.8:/home/your_account/`
> 
> `scp ./file pi:/home/your_account/`

그 다음으로 엔진엑스(Nginx)가 설치되어 있는지 확인해봅니다. `nginx -v` 명령을 실행합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-09.png)

위와 같은 결과가 나온다면 엔진엑스가 설치되어 있지 않은겁니다. `sudo apt install nginx` 를 실행하여 엔진엑스를 설치해줍니다.

