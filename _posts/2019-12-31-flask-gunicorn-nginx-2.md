---
title: 'Flask + Gunicorn + Nginx (2) Gunicorn'
date: 2019-12-31 16:55:00 -0400
categories: Flask
---

이전 단계에서 Nginx 서버를 구동하는 방법을 다뤘고, 이번 포스트에서는 `Gunicorn`을 설정해보고자 한다. 맨 아래에 걸어둔 링크를 참조하여 실습 및 문서를 작성했다.

그 동안 작업해 온 프로젝트로 테스트해보려 했지만 EC2에서는 컨테이너 단위로만 배포하고 있었기 때문에 괜한 삽질이 너무 많아질 것 같아 튜토리얼의 간단한 샘플 프로젝트를 따라 진행하기로 했다.

```python
# myproject.py

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return '<h1 style="color: blue">하이하이하이하이</h1>'

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```



현재까지는 아직 `flask run`으로 서버를 돌릴 수 있는 환경이 아니기 때문에 `python` 인터프리터를 사용한다. 프로덕션 환경에서는 WSGI 서버를 사용하라는 경고도 함께 뜬다. 왠지 반갑다.

```
(venv) python myproject.py

 * Serving Flask app "myproject" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```



<img width="614" alt="image" src="https://user-images.githubusercontent.com/12066892/71613666-27026a00-2beb-11ea-9929-f791aee6be4d.png">



### WSGI 엔트리 포인트 생성

Flask 앱과 Gunicorn 서버 간 상호작용을 위한 엔트리 포인트 파일을 프로젝트 루트 디렉토리에 생성한다. Django는 프로젝트를 생성할 때 자동으로 파일을 만들어주지만 Flask에서는 수동으로 만들어줘야 한다.

```python
# wsgi.py

from flaboard import app

if __name__ == '__main__':
    app.run()
```

이전 Flask 프로젝트에서 사용했던`app.py` 파일과 굉장히 유사하다.



### Gunicorn 환경설정

`엔트리 포인트:Flask 앱 변수` 형식으로 Gunicorn 서버를 구동시킨다. 엔트리 포인트 이름에서 모듈의 확장자를 뺀 이름을 사용하고, `myproject.py`에서 정의한 Flask 앱의 변수명을 사용한다.

```
(venv) gunicorn --bind 0.0.0.0:5000 wsgi:app

[2019-12-31 07:40:40 +0000] [3080] [INFO] Starting gunicorn 20.0.4
[2019-12-31 07:40:40 +0000] [3080] [INFO] Listening at: http://0.0.0.0:5000 (3080)
[2019-12-31 07:40:40 +0000] [3080] [INFO] Using worker: sync
[2019-12-31 07:40:40 +0000] [3083] [INFO] Booting worker with pid: 3083
[2019-12-31 07:41:16 +0000] [3080] [CRITICAL] WORKER TIMEOUT (pid:3083)
[2019-12-31 07:41:16 +0000] [3083] [INFO] Worker exiting (pid: 3083)
[2019-12-31 07:41:16 +0000] [3085] [INFO] Booting worker with pid: 3085
```

`python myproject.py`로 구동했을 때와 동일하게 접속된다.



### Gunicorn 서버 자동 구동

`systemd` 서비스 유닛으로 할당하여 Ubuntu 시작과 동시에 Gunicorn 서버도 자동으로 시작할 수 있도록 설정해보자.

```
$ deactivate
$ cd /etc/systemd/system
$ sudo vim myproject.service
```

```
[Unit]
Description=Gunicorn instance to serve myproject
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myproject
Environment="PATH=/home/ubuntu/myproject/venv/bin"
ExecStart=/home/ubuntu/myproject/venv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m 007 wsgi:app
```

`Service` 섹션의 `User`는 현재 Ubuntu의 사용자 이름이고, Nginx 서버와 Gunicorn 서버 프로세스가 있도록 `www-data`로 그룹 오너십을 부여한다.

작업 디렉토리를 매핑하고 `PATH` 환경변수를 설정하여 가상 환경 내의 프로세스들이 실행 가능하다는 것을 인지시키자. 3개의 작업자 프로세스가 필요한데, `WorkingDirectory`, `Environment`, `ExecStart` 섹션이 아래 절차를 수행한다.



- 프로젝트 루트 디렉토리에 `myproject.sock`이라는 Unix 소켓 파일을 생성하고 바인딩한다.
- 외부 사용자의 접속을 제외한 오너와 오너 그룹만 허용하기 위해 `umask 007` 값을 세팅한다.
- `wsgi:app`으로 엔트리 포인트를 지정한다.



마지막으로 `[Install]` 섹션을 작성하여 OS를 부팅할 때 systemd가 이 서비스를 어디에 연결할지를 설정한다.

```
[Install]
WantedBy=multi-user.target
```



이제 `myproject`라는 이름으로 서비스를 실행할 수 있다.

```
$ sudo systemctl start myproject
$ sudo systemctl enable myproject
$ sudo systemctl status myproject

● myproject.service - Gunicorn instance to serve myproject
   Loaded: loaded (/etc/systemd/system/myproject.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2019-12-31 09:05:35 UTC; 17min ago
 Main PID: 3335 (gunicorn)
    Tasks: 4 (limit: 1152)
   CGroup: /system.slice/myproject.service
           ├─3335 /home/ubuntu/myproject/venv/bin/python3 /home/ubuntu/myproject/venv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m
           ├─3353 /home/ubuntu/myproject/venv/bin/python3 /home/ubuntu/myproject/venv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m
           ├─3355 /home/ubuntu/myproject/venv/bin/python3 /home/ubuntu/myproject/venv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m
           └─3357 /home/ubuntu/myproject/venv/bin/python3 /home/ubuntu/myproject/venv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m

Dec 31 09:05:35 ip-172-31-32-34 systemd[1]: Started Gunicorn instance to serve myproject.
Dec 31 09:05:35 ip-172-31-32-34 gunicorn[3335]: [2019-12-31 09:05:35 +0000] [3335] [INFO] Starting gunicorn 20.0.4
Dec 31 09:05:35 ip-172-31-32-34 gunicorn[3335]: [2019-12-31 09:05:35 +0000] [3335] [INFO] Listening at: unix:myproject.sock (3335)
Dec 31 09:05:35 ip-172-31-32-34 gunicorn[3335]: [2019-12-31 09:05:35 +0000] [3335] [INFO] Using worker: sync
Dec 31 09:05:35 ip-172-31-32-34 gunicorn[3335]: [2019-12-31 09:05:35 +0000] [3353] [INFO] Booting worker with pid: 3353
Dec 31 09:05:35 ip-172-31-32-34 gunicorn[3335]: [2019-12-31 09:05:35 +0000] [3355] [INFO] Booting worker with pid: 3355
Dec 31 09:05:35 ip-172-31-32-34 gunicorn[3335]: [2019-12-31 09:05:35 +0000] [3357] [INFO] Booting worker with pid: 3357
```



> https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-18-04