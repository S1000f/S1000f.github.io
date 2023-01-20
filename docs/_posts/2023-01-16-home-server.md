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

## 2.1 HTML 작성

HTTP 서버를 사용해서 매우 간단한 정적 HTML 페이지를 응답하는 서버를 설정하겠습니다. HTTP 서버는 nginx 를 사용합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-08.png)

테스트용 이므로 아주 간단한 HTML 문서를 작성했습니다(사실 프론트는 잘 모릅니다). 파이에서 바로 작성해도 되고 다른 컴퓨터에서 작성 후 FTP, 이동식 미디어장치 혹은 scp 명령 등으로
파일을 서버에 복사해줍니다.

> scp(secure copy) 사용법은 아래와 같습니다. 위에서 ssh 호스트 설정을 하셨다면 그 호스트별칭을 사용할 수 있습니다.
> 원격지의 복사 경로는 절대경로를 사용해주시고, 홈을 의미하는 '~' 는 사용하지 말아주세요.
>
> `scp ./file your_account@192.168.0.8:/home/your_account/`
> 
> `scp ./file pi:/home/your_account/`

## 2.2 Nginx 설정

그 다음으로 엔진엑스(Nginx)가 설치되어 있는지 확인해봅니다. `nginx -v` 명령을 실행합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-09.png)

위와 같은 결과가 나온다면 엔진엑스가 설치되어 있지 않은겁니다. `sudo apt install nginx` 를 실행하여 엔진엑스를 설치해줍니다.
혹은 엔진엑스 공식 사이트의 설치 가이드를 따라서 원하시는 특정 버전을 설치해주시면 됩니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-11.png)

설치가 정상적으로 완료되었거나 이미 설치되어 있었다면 다음과 같이 엔진엑스 버전이 정상적으로 출력됩니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-12.png)

엔진엑스는 `/etc/nginx` 경로에 있습니다. `sites-available` 과 `sites-enable` 에 가상 호스트 설정을 추가하여 우리가 원하는
프론트 서버와 백엔드 서버를 설정할 수 있습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-13.png)

`sites-available` 에 위치한 `default` 파일을 아무 에디터로 열어봅니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-14.png)

'server {...}' 안의 정보로 엔진엑스 가상 호스트를 생성할 수 있습니다.
22행의 `listen 80 default_server;` 을 보시면 이 서버는 80번 포트에 바인딩되는 것을 알 수 있습니다.
80번 포트로 HTTP 요청이 왔을경우 41행의 `root` '/var/www' 경로에 있는 `location` '/' 디렉터리 아래에서, `index` 에 열거된 이름의 파일이
있는지 확인합니다.

따라서 우리가 작성한 'index.html' 파일을 저 경로에 복사해도 되고, 아니면 우리가 원하는 경로를 `root` 에 적어주시면 됩니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/_posts/home-server-15.png)

위의 예시는 `root` 설정을 변경한 예시입니다. 여러분이 원하는 경로를 입력해주세요.

> Vim 에디터에서 위 사진처럼 행 번호가 보이지 않는다면 `:se nu` 타이핑 후 엔터를 누르면 행 번호가 보입니다.
> 
> 그 다음 `root` 가 적힌 행으로 이동하기 위해 `41G` 타이핑 후 엔터를 누릅니다.
> 
> 그 다음 `w` 를 누르면 커서가 /var/www 바로 앞으로 이동합니다.
> 
> 그 다음 `cW` 를 누르면(W 는 대문자 이므로 쉬프트키를 누르고 입력) 기존에 있던 내용이 삭제됩니다.
> 
> 그 다음 원하시는 경로를 다 입력하신 후 esc 키를 눌러줍니다.
> 
> 마지막으로 `:wq` 타이핑 후 엔터를 누르면 편집된 결과를 저장하고 에디터에서 빠져나옵니다.

만약 편집을 하셨다면 변경된 내용이 엔진엑스에 반영되도록 `sudo nginx -s reload` 명령을 실행해줍니다.
실행했을때 만약 에러메시지가 나타난다면 편집한 내용중 오류가 있는 경우입니다. 보통은 설정구문 마지막에 세미콜론(;) 을 빠뜨리거나 띄어쓰기가 잘못된 경우입니다.
저는 특히 세미콜론을 자주 깜빡합니다. 엔진엑스 리로드에 실패했다면 다시 파일을 열어 제대로 편집이 되었는지 확인해봅니다.

## 2.3 포트포워딩 설정

저의 서버 아이피 192.168.0.8 는 저희집 내부에서만 사용되는 사설 네트워크의 서브넷 아이피입니다. 외부에서 우리 서버에 HTTP 요청을 보내고 우리 서버가
응답을 하기 위해선 고유한 public IP 가 필요합니다. 가정집에선 대부분 유동 아이피를 사용하지만 일단은 그 아이피를 그대로 사용하겠습니다.

현재 우리집에 할당된(ISP 혹은 지역사업자가 할당해준) 아이피를 확인하기 위해서 서버에서 `curl ifconfig.me` 명령어를 실행합니다.
혹은 구글에서 'my public ip' 로 검색한 결과의 사이트에서 나의 퍼블릭 아이피를 알 수 있습니다.