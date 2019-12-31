---
title: 'Flask + Gunicorn + Nginx (1) Nginx 서버'
date: 2019-12-31 17:00:00 -0400
categories: Flask
---

# WSGI

## Gunicorn

`Green Unicorn`의 줄임말로서 Ruby 진영의 `Unicorn`에서 포팅된 Unix 전용 파이썬 WSGI HTTP 서버이다. Gunicorn 팀에서 `Nginx`와 함께 사용하도록 권장하고 있다.

8000번 포트로 구동되고 의존성이 없으며, Nginx는 일반적으로 리버스 프록시 서버로 사용된다. 

- Paster, Django, WSGI와 사용
- 워커 프로세스 관리 자동화
- 파이썬 설정 쉬움
- 여러 워커 설정 사용 가능
- 서버 hook의 다양성

동기적인 워커들은 Nginx 뒤에서 실행되도록 빌드되었고, Nginx 업스트림 서버는 HTTP/1.0.만을 사용한다.

> **리버스 프록시 서버(Reverse Proxy Server)**
>
> 서버에서 클라이언트 방향으로 데이터를 전달하는 포워드 프록시 서버(Forward Proxy Server)와 상반되는 개념이다. 다수의 서버가 존재하고 매 요청 발생 시 어떤 서버에게 이 요청을 처리할지 지시하는 역할을 한다.
>
> 리버스 프록시 서버와 포워드 프록시 서버를 하나의 프록시 서버로 운용할 수 있지만 여의치 않을 때는 구분하기도 한다.



```
(venv) $ pip install gunicorn
```



## Flask + Gunicorn + Nginx

WSGI인 Gunicorn과 리버스 프록시 서버 역할을 할 Nginx, 그리고 Flask 프레임워크로 개발한 앱을 구성하여 배포해보자.

### 테스트 환경

Ubuntu 18.04

Python 3.6.9

Docker 18.09.7

MySQL 5.7

Nginx 1.14.0



---

### Nginx 설치

```
$ sudo apt-get install -y nginx
```



### 방화벽 해제

Nginx를 설치하는 과정에서 스스로 `ufw(Ubuntu Firewall)`에 등록하고, Nginx 서버에 접근할 수 있도록 방화벽을 해제하자.

```
$ sudo ufw app list

Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

- **Nginx Full:** 80번 포트와 443번 포트를 둘 다 연다.
- **Nginx HTTP:** 80번 포트만 연다.
- **Nginx HTTPS:** 443번 포트만 연다.



HTTPS에 대한 설정을 하지 않았기 때문에 일단 `Nginx HTTP` 프로파일로 80번 포트만 열자.

```
$ sudo ufw allow 'Nginx HTTP'

Rules updated
Rules updated (v6)
```

잘 열려 있는지 아래 커맨드로 확인할 수 있는데 아직 비활성화 상태이다.

```
$ sudo ufw status

Status: inactive
```

찾아보니 ufw가 아직 enable로 활성화를 시키지 않아서 그런 것이었다. ㅎㅎ

```
$ sudo ufw enable

Command may disrupt existing ssh connections. Proceed with operation (y|n)?
Firewall is active and enabled on system startup
```

다시 상태를 조회해보면 아까 허용했던 80번 포트가 추가되어있는 것을 확인할 수 있다.

```
$ sudo ufw status

Status: active

To                         Action      From
--                         ------      ----
Nginx HTTP                 ALLOW       Anywhere
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
```

포트도 허용했고 Nginx 서버가 잘 돌아가고 있는지 `systemctl` 커맨드로 확인하자.

```
$ systemctl status nginx

● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2019-12-31 02:39:34 UTC; 21min ago
     Docs: man:nginx(8)
 Main PID: 18788 (nginx)
    Tasks: 2 (limit: 1152)
   CGroup: /system.slice/nginx.service
           ├─18788 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           └─18790 nginx: worker process
           
...
```

위 방법으로도 확인이 가능하지만 직접 서버 IP로 접속하여 아래처럼 Nginx 기본 페이지가 잘 뜨는지 확인하는 것이 사실 제일 정확하다고 할 수 있다.

<img width="569" alt="image" src="https://user-images.githubusercontent.com/12066892/71608668-328f6a00-2bc6-11ea-940c-f9db8a34a88f.png">

위에서 ufw를 활성화하면서 ssh 접속이 끊어질 수 있다는 메시지에 y라고 응답했었다. 당시에는 아무 이상이 없었으나 점심먹고 와서 당연하게 끊어진 EC2를 다시 접속해보니 ssh 타임아웃 에러가 나면서 접속이 되지 않는다. 인스턴스 재부팅도 안 먹고, 인바운드 재설정도 먹지 않는다.

```
ssh: connect to host 52.79.237.12 port 22: Operation timed out
```



### Nginx 프로세스 관리

Ubuntu에서 프로세스를 관리하는 커맨드들을 알아보자. 대부분 낯이 익다.

```
$ sudo systemctl stop nginx
$ sudo systemctl start nginx
$ sudo systemctl restart nginx
$ sudo systemctl reload nginx
	일부 설정이 변경되었을 때 stop 시키지 않고 간단하게 재구동
$ sudo systemctl disable nginx
	서버 부팅 시 Nginx 자동 실행 비활성화
$ sudo systemctl enable nginx
	서버 부팅 시 Nginx 자동 실행 활성화
```



### 서버 블록(Server Block) 설정

`서버 블록`은 Apache 서버의 `VirtualHost`와 동일한 것이며, TCP 소켓에 바인딩하기 위해 `server_name`과 `listen` 커맨드를 사용한다.

아래 코드에서 알 수 있듯이 한 대의 서버 안에 여러 대의 서버가 존재하는 것처럼 가상의 호스트를 사용하는 것이다. `www.domain1.com`과 `www.domain2.com` URL 모두 동일한 IP를 바라보고 있지만 각 도메인에 따라 다른 페이지에 접속할 수 있도록 분기한다.

```
http {
  index index.html;

  server {
    server_name www.domain1.com;
    access_log logs/domain1.access.log main;

    root /var/www/domain1.com/htdocs;
  }

  server {
    server_name www.domain2.com;
    access_log  logs/domain2.access.log main;

    root /var/www/domain2.com/htdocs;
  }
}
```



Nginx는 기본적으로 `/var/www/html` 디렉토리를 서비스하는 서버 블록이 1개만 설정되어 있다. 싱글 사이트를 서비스하는 경우에는 상관없겠지만 2개 이상의 사이트를 서비스하는 경우에는 이야기가 달라진다. `/var/www/html` 경로 사이에 `example.com` 디렉토리를 추가하자. `p` 옵션으로 하위 디렉토리까지 함께 생성할 수 있다.

```
$ sudo mkdir -p /var/www/example.com/html
```



생성 직후의 오너십이 `root`로 되어 있기 때문에 일반 사용자로 변경해준다.

```
$ sudo chmod -R $USER:$USER /var/www/example.com/html
```



이제 `html` 디렉토리에 `index.html` 파일을 생성한다.

```
$ cd /var/www/example.com/html
$ vim index.html
```

```html
<html>
    <head>
        <title>첫 번째 서버 블록</title>
    </head>
    <body>
        <h1>첫 번째 서버 블록 접속 성공</h1>
    </body>
</html>
```



Nginx 서버를 통해 방금 만든 정적 파일을 제공하려면 서버 블록 설정 파일을 작성해야 한다. 기존의 `default` 파일을 수정할 수도 있지만 새로 만들어서 해보자. 서버의 설정 파일을 건드리는 것이므로 루트 권한이 필요하다.

```
$ cd /etc/nginx/sites-available
$ sudo vim example.com
```

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/example.com/html;
        index index.html index.htm index.nginx-debian.html;

        server_name example.com www.example.com;

        location / {
                try_files $uri $uri/ =404;
        }
}
```



`root` 부분을 우리가 제공하는 정적 파일의 경로로 수정했고, `server_name`은 `example.com`으로 입력했다. Nginx가 시작할 때 읽어오는 `sites-enabled` 디렉토리에 링크를 걸어주자. 링크의 원천은 절대경로로 입력해야 한다.

```
$ sudo ln -s /etc/nginx/siets-available/example.com ../sites-enabled/
$ ll
```

```
total 8
drwxr-xr-x 2 root root 4096 Dec 31 06:29 ./
drwxr-xr-x 8 root root 4096 Dec 31 02:39 ../
lrwxrwxrwx 1 root root   34 Dec 31 02:39 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root   38 Dec 31 06:29 example.com -> /etc/nginx/sites-available/example.com
```



이제 `example.com`과 `www.example.com`으로 접속할 수 있는 2개의 서버 블록을 구성했다. 이 두 URL로 접속하면 아까 새로 생성한 index.html을 응답받고, 80번 포트에 대한 나머지 요청은 모두 `default`에 설정된 `/var/www/html` 경로 내의 컨텐츠들을 응답받는다.

server_name을 새로 추가하면서 발생할 수 있는 `해시 버킷 메모리 문제(Hash Bucket Memory Problem)`를 방지하기 위해 `nginx.conf` 파일을 일부 수정해준다. 마찬가지로 루트 권한으로 작업한다.

```
$ cd /etc/nginx
$ sudo vim nginx.conf
```

```
# server_names_hash_bucket_size 64;

# 주석 해제
server_names_hash_bucket_size 64;
```



수정한 config 파일의 문법이 올바른지 테스트하고, Nginx를 재부팅한다.

```
$ sudo nginx -t

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

```
$ sudo systemctl restart nginx
```



### Nginx 주요 파일 및 디렉토리

#### 컨텐츠

- **/var/www/html:** Nginx가 기본적으로 서비스하는 컨텐츠들의 위치이다.

#### 서버 설정

- **/etc/nginx:** Nginx 설정 파일들이 있는 디렉토리이다.
- **/etc/nginx/nginx.conf:** Nginx의 주요 설정 파일로서, 글로벌 환경변수들을 설정한다.
- **/etc/nginx/sites-available:** 사이트별 서버 블록이 저장되는 디렉토리이다. Nginx는 이 안에서 직접 서버 블록을 찾지 않고 symbolic link로 연결시킨 `sites-enabled` 디렉토리에서 서버 블록을 찾게 된다.
- **/etc/nginx/sites-enabled:** 활성화된 사이트별 서버 블록이 저장되는 디렉토리이다.
- **/etc/nginx/snippets:** Nginx 환경설정에 사용될 설정 조각들이 있는 디렉토리이다. 반복적인 설정의 경우 이 디렉토리를 참조하면 간편하다.

#### 서버 로그

- **/var/log/nginx/access.log:** 별다른 로그 파일 경로 설정을 하지 않는다면 모든 요청에 대한 로그가 이 곳에 기록된다.
- **/var/log/nginx/error.log:** 에러 로그들이 기록되는 디렉토리이다.



> - https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-18-04
>
> - https://www.nginx.com/resources/wiki/start/topics/examples/server_blocks/
> - https://opentutorials.org/module/384/4529

