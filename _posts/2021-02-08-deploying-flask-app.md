---
layout: post
toc: true
title: "Nginx와 Gunicorn으로 Flask앱 배포하기"
categories: 서버개발
tags: [nginx, gunicorn, flask]
author:
  - jimin-mjl
---

## 1. 참고 자료
- [nginx 설치](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04)
- [gunicorn 설치 및 nginx와 연결](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-18-04)
- [우분투 서버에서 flask앱 배포하기](https://youtu.be/kDRRtPO0YPA) 

처음에 세 번째의 영상을 보고 따라했는데, 연결이 잘 되는 것을 확인했지만 gunicorn에서 빠져나왔을 때 서버가 멈춘다. 내가 원하는 건 계속 돌아가게 하는 거였기 때문에 이 영상을 통해서는 연결이 잘 되는지 확인한 다음, 위의 두 글을 통해 이후의 단계를 진행했다. 

***
## 2. 들어가기 전에 
처음에는 nginx와 uwsgi를 사용하려고 했는데, uwsgi를 설치하는데 계속 에러가 나서 gunicorn으로 시도해보았다. (uwsgi는 어떻게 읽는 걸까?)

모르는 게 너무 많아서 아직 이해를 다 못한 부분도 많지만 일단 이해한 만큼 적어보았다. 
### 2-1. WSGI란? 
CGI(Common Gateway Interface)의 일종으로, 프레임워크의 웹 서버이다. 

WSGI에는 두 종류가 있다.
- web server : Nginx, Apache
- web app : 파이썬 스크립트 파일

그럼 Gunicorn, uWSGI은? web server와 web app의 middleware라고 할 수 있다. 

정리하자면 이런 식이다. 
- 내가 만든 앱 : flask app
- 웹 서버 : nginx
- 앱과 웹 서버를 연결하는 미들웨어: gunicorn

#### [참고 글](https://paphopu.tistory.com/entry/WSGI%EC%97%90-%EB%8C%80%ED%95%9C-%EC%84%A4%EB%AA%85-WSGI%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80) 

### 2-2. 전체 프로세스
> client request -> nginx : web proxy server -> gunicorn : application server -> flask app

서버가 응답할 때는 요청 받은 순서 반대로 수행한다. 
참고로 proxy server는  클라이언트와 서버를 중계하는 서버이다. 

***
## 3. Nginx
### 3-1. Nginx 설치하기
```zsh
sudo apt update 
sudo apt install nginx 
```
### 3-2. 방화벽 설정하기
nginx의 접근을 허용하기 위해 방화벽 설정부터 해줘야 한다. 
우선 `sudo ufw app list` 를 통해 설정 가능한 방화벽을 확인하면 다음과 같이 나온다.
```
Available applications:
  Nginx Full    // HTTP, HTTPS 모두 허용
  Nginx HTTP    // HTTP만 허용
  Nginx HTTPS   // HTTPS만 허용
  OpenSSH
```
설정은 `sudo ufw allow 'Nginx HTTP'` 를 통해 가능하고,
`sudo ufw status` 를 통해 확인할 수 있다. 
### 3-3. 웹 서버 확인하기
nginx가 설치되면 자동으로 실행된다. 
상태를 확인하고 싶다면 `systemctl status nginx` 로 가능하다. 

그런데, 내 경우에는 이게 inactive 상태였다. 
그래서 `sudo ufw enable` 을 추가적으로 입력해서 active상태로 만들었다.
### 3-4. 부록 : Nginx 상태 관련 명령어 정리
```zsh
sudo systemctl start nginx   // nginx 시작
sudo systemctl stop nginx   // nginx 멈추기
sudo systemctl restart nginx   // nginx 재시작(멈추고 다시 시작)
sudo systemctl reload nginx   // 설정 파일만 수정했을 경우 연결을 끊지 않고 수정사항 적용시키기
sudo systemctl disable nginx   // nginx는 서버가 부팅되면 자동으로 시작된다. 이 설정을 지우기
sudo systemctl enable nginx   // 위의 설정을 다시 enable 시키기
```
***
## 4. Gunicorn
### 4-1. 설치 전에 의존 패키지 설치하기
```zsh
sudo apt update
sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools
```
### 4-2. 가상환경 설치하고 접속하기
```zsh
sudo apt install python3-venv
python3 -m venv venv-name
source venv-name/bin/activate
```
이제 가상환경 안에서 작업한다. 
### 4-3. Gunicorn 설치하기
```zsh
pip install wheel // package에 wheel archive가 없어도 설치되게끔 하기 위해서라는데 일단 설치한다. 
pip install gunicorn flask
which gunicorn // gunicorn 경로 확인하기
```
wheel 같은 경우는 uwsgi를 설치하려고 했을 때 uwsgi를 위한 wheel이 없다는 에러 메세지가 계속 뜨던데 그런 걸 말하는 것 같다.
### 4-4. Entry Point 파일 만들기
app을 import한 wsgi.py 파일을 만들어줬다. app.py를 그냥 entry point로 써도 되지 않을까 싶어서 처음에는 따로 만들지 않았는데 나중에 에러가 나서 찾아보다가 django는 wsgi.py파일을 찾아본다는 소리를 들었다. 근데 이건 flask 앱인데 이 문제가 맞을까 싶었지만 혹시 모르니까 일단 만들어줬다. 

만들고 나서 실행되는지 체크해보기 : `gunicorn wsgi:app`
wsgi는 파일 이름이고 app은 app 이름이다. 

### 4-5. service 파일 만들기
가상환경에서 빠져나와서 작업한다. 
빠져나오는 명령어는 `deactivate`.
#### 1) /etc/systemd/system/ 디렉토리 안에 /etc/systemd/system/myflask.service 파일을 만들어준다. 
```zsh
sudo nano /etc/systemd/system/myproject.service   // 참고로 etc 앞에 /가 안붙으면 새로운 파일일 경우 알아서 만들어주지 않으니 붙여준다.
```
파일 이름은 아무거나 해도 상관없지만 되도록 다른 파일 이름이나 디렉토리 이름이랑 통일시켜주자...! 헷갈려 죽겠다.
여기서는 구분을 위해 파일 이름은 myflask, 프로젝트 디렉토리 이름은 myproject, 유저 이름은 userme로 했다. 

#### 2) 다음의 내용을 작성한다. 
```
[Unit]
Description=Gunicorn instance to serve myflask
After=network.target

[Service]
User=userme
Group=www-data
WorkingDirectory=/home/userme/myproject
Environment="PATH=/home/userme/myproject/myproject-env/bin"
ExecStart=/home/userme/myproject/myproject-env/bin/gunicorn --workers 3 --bind unix:myflask.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target
```
unix:myflask.sock 부분이 nginx와 연결될 socket 파일이다. 
#### 3) service 파일 실행시키고 상태 확인하기
```zsh
sudo systemctl start myflask
sudo systemctl enable myflask
sudo systemctl status myflask
```
#### 4) 오류 해결하기
위의 예시 그대로 했더니 상태가 failed(result:exit-code) 로 뜨길래,
service 파일의 `ExecStart= --workers 3` 
이 부분을 `--workers 1` 으로 바꿔준 다음,
`sudo systemctl restart myflask` 로 재시작했더니 잘 작동하는 것을 확인했다. 

***

## 5. Gunicorn을 Nginx와 연결시키기
### 5-1. 들어가기 전에
우선 nginx의 디렉토리 구조를 이해할 필요가 있다.
나도 자세히는 이해하지 못했기 때문에 우선 이해한 만큼만 적어본다.
여기서는 두 개만 알면 된다.
- /etc/nginx/sites-available/ : 여기서 설정 파일을 만들어서 수정한다.
- /etc/nginx/sites-enabled/ : sited-available 디렉토리의 설정 파일이 이 디렉토리의 파일과 link되어 nginx에 적용된다.

정리하자면, 설정파일을 우리가 생성하고 수정할 때는 sites-available에 있는 파일을 건드리고 sites-enabled의 파일은 직접 건드리지 않는다.
그런데 nginx는 sites-enabled를 찾아보지 sites-available을 찾아보지 않는다.
그래서 sites-available에서 수정한 파일을 sites-enable의 파일과 링크시켜서 수정 사항이 반영되도록 한다.
(그런데 참고한 영상에서는 바로 sites-enabled의 파일을 수정하던데 별 상관 없는건가 싶기도 하다)

### 5-2. nginx의 설정 파일 생성하기
/etc/nginx/sites-available/에 새 설정 파일을 만들어준다. 
#### a. 기존의 파일(default)을 삭제하고 내 것(myflask)으로 다시 만들어준다.
```zsh
sudo rm /etc/nginx/sites-available/default
sudo rm /etc/nginx/sites-enabled/default
sudo touch /etc/nginx/sites-available/myflask
```
#### b. 에디터로 열어서 다음의 내용을 작성하고 저장한다.
`sudo nano /etc/nginx/sites-available/myflask` 로 파일을 열어준다. 
```
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name 0.0.0.0;
	location / {
		include proxy_params;
		proxy_pass http://unix:/home/sammy/myproject/myproject.sock; // gunicorn service 파일에서 socket 파일의 경로
	}
}
```
내 경우 proxy_pass는 `http://unix:/home/userme/myproject/myflask.sock` 이다.
### 5-3. sites-available의 파일과 sites-enabled의 파일을 연결시키기
두 디렉토리의 서로 다른 파일 사이에 심볼릭 링크를 생성해서 연결시키는 작업을 해준다.
```zsh
sudo ln -s /etc/nginx/sites-available/myflask /etc/nginx/sites-enabled/myflask    
```
ln : link, -s : symbolic
### 5-4. 마무리 작업
마지막으로 해당 설정 파일에 문법 오류가 없는지 체크하고, 아무 문제가 없으면 nginx를 재시작한다.
```zsh
sudo nginx -t  // 문법 오류 체크
sudo systemctl restart nginx
```
### 5-5. 접속해서 확인하기
이제 원격 IP 주소로 접속해서 잘 뜨는지 확인하면 끝!
### 5-6. 정상 작동하지 않는다면
만약 여기서 잘 안되면 방화벽을 확인해보자. 가상머신의 네트워킹도 확인해보면서 이것저것 시도해보자! 
```zsh
sudo ufw app list
sudo ufw allow 'Nginx Full'  // HTTP, HTTPS 모두 접근 허용
sudo ufw status
```
