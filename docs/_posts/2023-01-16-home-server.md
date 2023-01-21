---
title: 집에있는 컴퓨터로 웹 서비스 배포 준비하기
published: true
tags: network, nginx
---

향후 완성될(언제가 될지 모르는) 토이 프로젝트 웹 서비스 배포를 준비하기 위해 집에있는 컴퓨터를 서버로 사용해 보겠습니다.
예제에서는 매우 간단한 프론트와 백엔드 데모 프로그램을 배포하겠습니다. 그리고 포스팅을 위해 일회성으로 도메인도 구입하여 연결 해보겠습니다.

---

# 1. 서버 준비

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-01.jpeg)

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
> ![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-02.png)

저는 아이피타임 공유기를 사용중입니다. 공유기가 DHCP 서버 역할을 하므로, 방금 연결된 기기에 알맞은 서브넷 IP 를 할당했을 겁니다.
아이피타임은 브라우저로 접속할 수 있는 GUI 설정을 지원합니다. 웹 브라우저를 열고 'http://192.168.0.1' 으로 접속하여 관리자 계정으로
로그인 합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-03.png)

메뉴 '고급 설정 -> 네트워크 관리 -> 내부 네트워크 설정' 에서 방금 연결한 파이의 아이피를 확인 할 수 있습니다.
저는 192.168.0.8 를 할당 받았습니다. 아이피 옆에 있는 데이터는 MAC 주소입니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-04.png)

혹은 위와 같이 `ifconfig` 명령을 통해 파이 터미널에서 아이피를 직접 확인하실 수 도 있습니다.

## 1.3 SSH 연결

만약 라즈베리 파이에 모니터가 연결되어 있어서 항상 파이의 터미널을 사용할 수 있다면 ssh(secure shell) 사용은 필수는 아닙니다.
저는 파이는 전원과 와이파이만 연결되어있고 별도의 모니터는 연결되어 있지 않으므로, ssh 접속을 사용하겠습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-05.png)

먼저 파이에서 `systemctl status ssh` 명령을 사용하여 ssh 데몬이 정상적으로 실행 중인지 확인합니다. 대부분은 기본적으로 부팅시
ssh 서비스가 실행되지만 어떤 배포판에서는 기본적으로 비활성화 되어 있는 경우를 봤습니다.
만약 ssh 서비스가 inactive 라면 `systemctl restart ssh` 명령으로 실행시켜줍니다.

이제 라즈베리 파이 외부의 원격지에서 ssh 로 접속하겠습니다.
`ssh <your_account>@192.168.0.8` 명령으로 접속합니다. <your_account> 부분을 여러분의 우분투 계정이름으로 바꿔주세요.
마찬가지로 아이피도 여러분의 서버의 서브넷 아이피로 바꿔주십시오. 접속시 해당 계정의 비밀번호를 입력합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-06.png)

ssh 로 접속하였습니다. 마지막 줄의 from 192.168.0.5 는 저의 데스크탑 컴퓨터의 서브넷 아이피입니다.
다음으로 매번 'ssh 계정@호스트' 를 타이핑 하기 귀찬으니 ssh 설정파일에 호스트를 등록하겠습니다.
~/.ssh/ 로 이동해서 'config' 파일이 있다면 수정하고 없다면 생성해줍니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-07.png)

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

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-08.png)

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

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-09.png)

위와 같은 결과가 나온다면 엔진엑스가 설치되어 있지 않은겁니다. `sudo apt install nginx` 를 실행하여 엔진엑스를 설치해줍니다.
혹은 엔진엑스 공식 사이트의 설치 가이드를 따라서 원하시는 특정 버전을 설치해주시면 됩니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-11.png)

설치가 정상적으로 완료되었거나 이미 설치되어 있었다면 다음과 같이 엔진엑스 버전이 정상적으로 출력됩니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-12.png)

엔진엑스는 `/etc/nginx` 경로에 있습니다. `sites-available` 과 `sites-enable` 에 가상 호스트 설정을 추가하여 우리가 원하는
프론트 서버와 백엔드 서버를 설정할 수 있습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-13.png)

`sites-available` 에 위치한 `default` 파일을 아무 에디터로 열어봅니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-14.png)

'server {...}' 안의 정보로 엔진엑스 가상 호스트를 생성할 수 있습니다.
22행의 `listen 80 default_server;` 을 보시면 이 서버는 80번 포트에 바인딩되는 것을 알 수 있습니다.
80번 포트로 HTTP 요청이 왔을경우 41행의 `root` '/var/www' 경로에 있는 `location` '/' 디렉터리 아래에서, `index` 에 열거된 이름의 파일이
있는지 확인합니다.

따라서 우리가 작성한 'index.html' 파일을 저 경로에 복사해도 되고, 아니면 우리가 원하는 경로를 `root` 에 적어주시면 됩니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-15.png)

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
혹은 구글에서 'my public ip' 로 검색한 결과의 사이트에 접속하거나 공유기의 관리자로 로그인하면 알 수 있습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-16.png)

포트포워딩 설정을 위해서 공유기 설정에 접속합니다. 아이피타임 공유기의 경우 '고급설정 -> NAT/라우터 관리 -> 포트포워드 설정' 으로 이동합니다.
내부 IP주소에 서버의 서브넷 아이피를 기입합니다. HTTP 는 TCP 프로토콜을 사용하므로 'TCP' 를 선택하고 외부포트와 내부 포트 모두 80으로 설정합니다.
내부 포트를 80으로 설정하는 이유는 2.2 파트에서 엔진엑스의 프론트 서버를 80번 포트를 리스닝 하도록 설정했기 때문입니다.

이 규칙을 적용하고 저장을 합니다. 그리고 웹 브라우저를 열고 위에서 확인한 우리집의 public IP 로 접속합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-17.png)

저는 처음에 이런 URI 로 리다이렉트 되며 엔진엑스가 파일을 찾을 수 없다는 404 에러를 응답했습니다.
이런 경우가 발생한다면 브라우저의 캐시를 비우고 새로고침을 해줍니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-18.png)

각자 작성한 정적 HTML 페이지가 잘 보인다면 정상적인 요청과 응답이 이뤄진겁니다. 이제 외부망에서도 우리집의 공용 아이피로 HTTP 요청시 엔진엑스가
적절한 응답을 하게 됩니다.

# 3. 백엔드 배포

이제 데모 역할을 해줄 매우 간단한 백엔드 웹 어플리케이션 서버를 만들어서 서버에서 실행해줍니다. 저는 코틀린의 `Ktor` 웹 프레임워크를 사용했습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-19.png)

내부적으로 `Netty` 서버를 사용하는 이 백엔드 API 서버는 8080 포트에 바인딩 됩니다.
루트 경로에 GET 요청시 마다 'hello, world!' 혹은 'good bye, world!' 를 JSON 형식에 담아 응답합니다.
CORS 호스트 설정은 도메인 연결 후 하도록 하겠습니다.

> 'good bye, world!' 는 C 언어의 창시자이자 UNIX 를 개발한 핵심 프로그래머 중 한명이었던 데니스 리치가 2011년 10월 12일 세상을 떠났을때,
> 그를 추모하기 위해 사용된 태그입니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-20.png)

홈서버에서 백엔드 API 서버를 가동해줍니다. `netstat -lptn` 명령을 실행하여 백엔드 서버가 정상적으로 리스닝 중인지 확인합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-21.png)

프론트 정적페이지 경우와 마찬가지로 8080 포트를 포워딩하여 홈서버의 8080 포트로 연결되도록 설정해줍니다. 저장을 누르고 나옵니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-22.png)

이제 웹 브라우저에서 '나의아이피:8080' 으로 접속해봅니다. 제가 작성한대로 응답결과를 잘 확인할 수 있습니다.

> 위의 사진과 같이 웹 브라우저에서 JSON 형식을 보기 쉽게 만들어주는 'JSON Viewer' 같은 크롬 확장프로그램을 사용하면 편합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-23.png)

프론트 파일을 위와 같이 수정했습니다. Fetch API 를 사용하여 백엔드에서 받은 데이터를 간단하게 보여줍니다.(사실 프론트엔드를 잘 모릅니다..)

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-24.png)

이제 웹브라우저에서 8080 포트번호를 지우고 80 포트로 접속하여 프론트 페이지를 요청해봅니다. API 서버와 통신이 잘 되고 있음을 확인합니다.

# 4. 도메인 설정

이제 우리 서비스를 위한 도메인을 구매 후 연결하겠습니다. 저는 GoDaddy 라는 서비스를 사용했습니다. 이 사이트 외에도 국내외 다른 서비스들이 있습니다.
무엇을 사용하든 상관은 없습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-25.png)

제 이름과 비슷한 도메인을 하나 정했는데 도대체 무슨 이유로 이게 '전 세계적 인기 항목' 인지는 모르겠습니다. 아마도 상술이겠지요.
그리고 탑레벨 도메인(TLD)이 com 인게 왜 최고의 솔루션인지도 모르겠습니다.
com 은 초기에 만들어진 전통적인 도메인이긴 하지만 그것이 왜 최고로 여겨지는진 모르겠습니다. 이것도 아마 상술이겠지요. 아무튼 가격이 싸니 구매해봅니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-26.png)
![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-27.png)

CNAME 별칭 레코드를 하나 추가합니다. 추후에 백엔드 API 프록시 서버 용도로 사용됩니다. 값은 구매한 도메인 그대로 입력합니다.
com 뒤에 마침표 . 은 루트 도메인을 의미합니다. 그리고 A 레코드의 값을 우리의 public IP 로 수정합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-28.png)

잠시후에 웹브라우저에서 구입한 도메인으로 접속을 해봅니다. DNS 서버에 우리가 구입한 도메인과 수정한 값이 반영되는데 시간이 소요될 수 도 있습니다.
DNS 서버들은 도메인과 매핑된 아이피 주소들을 기록하는 테이블을 캐싱합니다. 그 캐시 값이 갱신 될 때까지 소요되는 시간입니다.

# 5. 인증서 등록 및 리버스 프록시 설정

4번 항목까지만 진행되어도 서비스를 배포하고 사용 자체는(?) 할 수 있습니다. 하지만 우리 서비스에 접속할때마다 사용자들은 브라우저의 경고를 보게 됩니다.
민감한 정보가 암호화되지 않은채로 공용망(인터넷)을 자유롭게 떠다니는게 보안상 좋을리 없습니다.

HTTPS 프로토콜은 핸드셰이크 과정에서 암복호화에 사용될 키를 교환하며, 연결이 성립 후 이동되는 패킷은 해당 키로 암호화되어 전송됩니다.
HTTPS 프로토콜을 사용하는 서비스로 설정해보겠습니다. 

## 5.1 certbot 준비

HTTPS 연결 수립시 사용될 키는 우리가 직접 설정할 수도 있습니다. 하지만 인증서는 HTTPS 연결 시 사용되는 키를 공인된 기관에서 보증하는 것입니다.
인증서를 받기 위해선 공인된 기관에서 구입을 해야합니다. 하지만 우린 돈이 없으므로 공짜로 인증서를 발급해주는 서비스를 사용하겠습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-29.png)

구글에서 'certbot' 을 검색 후 해당 사이트에 접속합니다. 그럼 위와 같은 페이지를 볼 수 있는데 가운데쯤 있는 선택창에서 Nginx 와 Ubuntu 를
선택하면 다음 안내페이지로 이동합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-30.png)

'snapd' 를 설치하라고 하는데 우분투 20버전 배포판에는 이미 설치되어 있습니다. 'snapd' 설치를 확인 하신 후 3, 5, 6번의 명령어들을 차례로 실행합니다.

## 5.2 Nginx 가상 호스트 추가

다음으로 프론트를 위한 nginx 가상 호스트를 추가하겠습니다. 이 엔진엑스 설정을 바탕으로 certbot 이 자동으로 인증서를 발급하기 위한 프로세스를 수행 해줍니다.
`/etc/nginx/sites-available` 경로에 다음과 같은 파일을 아무 이름으로(예제에서는 front)생성해줍니다.

```
server {
    listen 80;
    server_name your_domain;
    root /home/your_account/app/front;
    index index.html;
}
```

`server_name` 필드에 여러분의 도메인을 입력해줍니다. 그외 사항은 기존의 default 설정 파일과 큰 차이는 없습니다.
저장 후 이 설정파일을 엔진엑스가 사용하도록 아래처럼 `/etc/nginx/sites-enable` 경로에 심볼릭 링크를 생성해줍니다. 

> `sudo ln -s /etc/nginx/sites-available/front /etc/nginx/sites-enable/`

그리고 잊지말고 `sudo nginx -s reload` 를 실행해줍니다.

## 5.3 인증서 발급

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-31.png)

`sudo certbot --nginx` 명령어를 실행합니다. 그럼 위의 사진처럼 1: 도메인명 항목을 볼 수 있습니다.
이 항목은 방금 작성한 엔진엑스 가상 호스트의 `server_name` 에 입력한 도메인명을 보여주고 있습니다.
1번을 선택 후 진행하면 성공적으로 인증서를 발급 받게 됩니다.

> 만약 인증서 발급에 실패한다면 다음 사항을 확인해봅니다.
> 
> 1. 엔진엑스 가상호스트 파일 작성 후, `sites-enable` 디렉터리에 심링크를 만들지 않았다.
> 2. 새로 구입한 도메인이 아직 DNS 서버에 반영이 되지 않았다.
> 3. 가상 호스트 설정 파일에 오타가 있다.
> 4. 80번 포트가 오픈되어있지 않거나 포트포워딩이 되어있지 않다.
> 
> 저는 주로 1번을 자주 깜빡하는 실수가 잦았습니다.

## 5.4 프록시 API 서버 설정

이제 인증서가 잘 적용되었는지 확인해보겠습니다. 우선은 다시 공유기 설정을 열어서 HTTPS 프로토콜이 사용하는 포트 443번을 포워딩해야합니다.
설정 방법은 이전 방식과 같습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-32.png)

웹 브라우저에서 우리 도메인으로 https:// 를 사용해서 접속하면 인증서가 잘 적용된 것을 확인 할 수 있습니다. 하지만 프론트에서 백엔드로의 ajax
통신은 위 콘솔 스크린샷과 같이 안됩니다. 이를 해결하기 위해 엔진엑스의 리버스 프록시 기능을 사용하겠습니다.

`/etc/nginx/sites-available` 경로에 다음과 같은 파일을 아무 이름으로(예제에서는 back)생성해줍니다.

```
server {
    server_name api.your_domain;
    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

`server_name` 필드에서는 이전의 4번 항목에서, DNS 설정 중에 API 서버를 위해 미리 추가했었던 CNAME 레코드를 기입합니다.
그리고 `location /` 블록에 `proxy_pass http://127.0.0.1:8080` 설정을 추가합니다.
엔진엑스가 api.your_domain 경로로 온 요청을 가로채 헤더를 수정 후 다시 백엔드 서버로 해당 요청을 전달합니다.

이전 프론트 가상 호스트 설정때와 마찬가지로 `sites-enable` 에 심볼릭 링크를 생성 해준 뒤 `sudo nginx -s reload` 를 실행하여 엔진엑스를 재시작합니다.
그리고 `sudo certbot --nginx` 명령을 실행 후 새로 추가한 백엔드 도메인을 선택하고 인증서 발급을 수행합니다.
마지막으로 프론트 페이지의 `Fetch API` 에서 기존의 'localhost:8080' 을 'api.your_domain' 으로 수정합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/home-server-34.png)

최종적으로 도메인과 인증서가 연결된 상태의 데모 서비스가 정상적으로 작동됨을 확인 할 수 있습니다.

## 6. 마무리

공유기의 포트포워드 설정에서 이제 불필요한 80번과 8080번 포워딩 설정을 삭제할 수 있습니다.
그리고 기존의 백엔드 소스에서 CORS 허용 호스트에 우리 도메인만을 지정하여 재배포 합니다.

이렇게 해서 우리집에서 홈서버로 개인 프로젝트를 서비스하기 위한 기본적인 준비사항을 알아보았습니다.
본문에 틀린 내용이 있거나 미흡한 부분이 있다면 댓글로 알려주시면 감사하겠습니다!

