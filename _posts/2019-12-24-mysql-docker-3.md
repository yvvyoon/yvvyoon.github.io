---
title: 'Dockerize MySQL (3)'
date: 2019-12-24 14:05:00 -0400
categories: Docker
---

내 로컬에서 EC2에 배포한 MySQL 커스텀 이미지에 정상적으로 붙는 것을 확인했다. 짝짝짝.
이번 포스트는 MySQL Dockerization 관련 마지막 포스트일 듯 한데, 또 다른 난관에 봉착했다.

Flask 컨테이너에서 MySQL 컨테이너로 붙질 않는다. 아래 에러 발생.

<br>

> _sqlalchemy.exc.OperationalError: (MySQLdb._exceptions.OperationalError) (1045, 'Plugin caching_sha2_password could not be loaded: /usr//usr/lib/x86_64-linux-gnu/mariadb19/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory')
(Background on this error at: http://sqlalche.me/e/e3q8)_

<br>

> *<참고>*
>
> *https://mysqlserverteam.com/mysql-8-0-4-new-default-authentication-plugin-caching_sha2_password/*

<br>

MySQL 8.0.4 버전부터 기본 인증 플러그인이 `mysql_native_password`에서 `caching_sha2_password`로 변경되었다. 로컬 MySQL도 8 버전인데 왜 에러가 없었나 했더니 `8.0.18`이어서 해당이 안된 것이었다. MySQL 이미지 만들 때 `Dockerfile`에 `mysql:8.0`이라고만 하고 그 뒷부분은 생략해서 최신 버전으로 당겨져버린 것이다..

구글링을 통해 찾은 해결법은, 아래처럼 Flask 내 MySQL 접속 정보 URI를 수정해주거나, MySQL 설정 파일을 일부 수정하면 된다.
하지만 잘 해결되지 않아 급한대로 필자는 5.7 버전으로 다운그레이드하여 해결했다.

<br>

> - SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'mysql://yoon:yoon@52.79.237.12:3306/flaboard?auth_plugin=myqsl_native_password'
>
> - Unix 환경에서는 `my.cnf`, Windows 환경에서는 `my.ini` 파일을 열어 보면 `default_authentication_plugin`의 값이 기본적으로 `caching_sha2_password`로 되어 있는데 `mysql_native_password`로 바꾸고 MySQL을 재실행해준다.

<br>

MySQL을 5.7 버전으로 다운그레이드하여 해결하다가 인코딩 관련 에러가 발생해서 그 부분까지 해결해야 정상적으로 MySQL 컨테이너에 붙을 수 있다.

<br>

앞 단계에서 생성한 `dump.sql` 파일을 열어서 기존에 `utf8mb4_0900_ai_ci`로 되어 있는 각 `COLLATE` 속성의 값을 `utf8mb4_unicode_ci`로 변경해준다.
전자는 8 버전에서의 utf8 인코딩, 후자는 5.7 버전에서의 utf8 인코딩이다.

<br>

```
...

DROP TABLE IF EXISTS `alembic_version`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `alembic_version` (
  `version_num` varchar(32) NOT NULL,
  PRIMARY KEY (`version_num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

...

```

<br>

`Dockerfile` 파일에서도 `FROM` 부분을 수정해준다.

```
FROM mysql:5.7 as builder

...
```
