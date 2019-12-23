---
title: 'Dockerize MySQL (2)'
date: 2019-12-23 16:29:00 -0400
categories: Docker
---

지난 포스트에 이어서 컨테이너가 중지되어도 데이터가 삭제되지 않게 volume을 설정하고, 데이터를 dump하여 Docker Hub에 push해보자. 다음 포스트에서는 EC2에서 커스텀 MySQL 이미지를 받아 테스트를 진행할 예정이다.

### Volume

일반적으로 컨테이너 내부에 데이터를 관리하므로, 컨테이너가 파기되면 그 안의 데이터들도 함께 파기되기 때문에 데이터베이스와 같은 컨테이너는 별도로 볼륨을 설정하여 데이터를 저장해야 한다. 아무리 컨테이너를 파기해도 데이터는 남아 있기 때문에 언제든지 접근할 수 있다.

`Dockerfile` 파일에서 `VOLUME` 옵션을 통해 경로를 지정해줄 수도 있고, `docker-compose.yml` 파일에 추가하는 방법으로 더욱 간단하게 적용할 수 있다.

-   docker-compose.yml

```yaml
version: '3'
services:
    web:
        build: .
        ports:
            - '5000:5000'
        volumes:
            - .:/code
            - logvolume01:/var/log
        links:
            - redis
    redis:
        image: redis
volumes:
    logvolume01: {}
```

`volumes`에 생성할 볼륨의 이름을 정하고 원하는 경로를 설정한다. `${volume_name}:${volume_path}` 형식으로 지정하면 된다.

docker-compose.yml에 설정해보기 전에 `-v` 옵션으로 볼륨을 지정해보자. OS 별로 경로가 다르기 때문에 주의하자. 아래 커맨드는 macOS 기준이다. 일반적으로 리눅스 환경에서는 `/var/lib/docker/` 경로로 지정하긴 하지만 결국 개인의 자유다.

필자는 별도의 디렉토리를 생성해서 설정할 것이다.

> _<참고>_
>
>  _https://medium.com/@crmcmullen/how-to-run-mysql-in-a-docker-container-on-macos-with-persistent-local-data-58b89aec496a_

```
$ docker run --name mysql-db -p 3306:3306 -d \
-e MYSQL_ROOT_PASSWORD=root \
-e MYSQL_USER=yoon \
-e MYSQL_PASSWORD=yoon \
-e MYSQL_DATABASE=flaboard \
-d mysql -v /Users/user/workspace/mysql-data:/var/lib/mysql
```

```
e265d1e7b550e160bde3133d65c06dafbe09e3b19fdad351ec2a93dc8836831f
```

### 데이터 덤프

Flask와 접속도 잘 되었고, 데이터도 샘플로 몇 건 넣었고, 이제 이미지 올릴 때 데이터도 같이 말아져 올라가는지 테스트해보자.
Flask와 같은 애플리케이션 레벨이라면 `Dockerfile` 파일을 만들어서 빌드하면 될텐데, MySQL은 어디에 만들어야 되는지 고민하다가 갑자기 머리가 하얘졌다.

구글링 하던 도중 나와 똑-같은 상황에 처해서 해결한 포스트를 발견했다.

> _<참고>_
>
> _https://dzone.com/articles/custom-mysql-docker-instance_

그 전에 반복 작업을 최소화하기 위해 shell 파일을 작성했다.

- start_mysql.sh

```shell
#!/bin/bash

echo '1. Remove the previous MySQL container.'
docker rm -f mysql-db

echo '2. Run the MySQL.'
docker run -d --name mysql-db -p 3306:3306 -d \
-e MYSQL_ROOT_PASSWORD=root \
-e MYSQL_USER=yoon \
-e MYSQL_PASSWORD=yoon \
-e MYSQL_DATABASE=flaboard \
-v /Users/user/workspace/mysql-data:/var/lib/mysql \
mysql

echo '3. Print containers list.'
docker ps
```

지금 하고자 하는 건 저장된 데이터까지 한꺼번에 이미지로 빌드하는 것이다. 그러므로 미리 데이터들을 dump해야 한다.
`mysqldump` 커맨드와 `--databases` 옵션으로 특정 데이터베이스의 데이터를 덤핑할 수 있다.

```
$ docker exec mysql-db /usr/bin/mysqldump -u root --password=root flaboard > dump.sql
```

와... 겨우 테이블 2개 뿐인데도 스크립트 길이가 어마어마하다.

```MySQL
-- MySQL dump 10.13  Distrib 8.0.18, for Linux (x86_64)
--
-- Host: localhost    Database: flaboard
-- ------------------------------------------------------
-- Server version	8.0.18

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!50503 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `alembic_version`
--

DROP TABLE IF EXISTS `alembic_version`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `alembic_version` (
  `version_num` varchar(32) NOT NULL,
  PRIMARY KEY (`version_num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `alembic_version`
--

LOCK TABLES `alembic_version` WRITE;
/*!40000 ALTER TABLE `alembic_version` DISABLE KEYS */;
INSERT INTO `alembic_version` VALUES ('ab88bf333048');
/*!40000 ALTER TABLE `alembic_version` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `article`
--

DROP TABLE IF EXISTS `article`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `article` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(100) DEFAULT NULL,
  `content` text,
  `created_datetime` datetime DEFAULT NULL,
  `author_id` int(11) DEFAULT NULL,
  `uploaded_file_path` varchar(200) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `author_id` (`author_id`),
  KEY `ix_article_created_datetime` (`created_datetime`),
  CONSTRAINT `article_ibfk_1` FOREIGN KEY (`author_id`) REFERENCES `user` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `article`
--

LOCK TABLES `article` WRITE;
/*!40000 ALTER TABLE `article` DISABLE KEYS */;
INSERT INTO `article` VALUES (1,'희희','희희','2019-12-23 15:04:47',1,NULL),(2,'덤프뜨자','덤프뜨자','2019-12-23 15:05:02',1,'None_1577081100_스크린샷 2019-11-27 오후 7.00.22.png'),(3,'갸아아아아아아악','ㄲㄱ','2019-12-23 15:05:28',2,NULL);
/*!40000 ALTER TABLE `article` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `comment`
--

DROP TABLE IF EXISTS `comment`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `comment` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `content` text,
  `created_datetime` datetime DEFAULT NULL,
  `article_id` int(11) DEFAULT NULL,
  `author_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `article_id` (`article_id`),
  KEY `author_id` (`author_id`),
  KEY `ix_comment_created_datetime` (`created_datetime`),
  CONSTRAINT `comment_ibfk_1` FOREIGN KEY (`article_id`) REFERENCES `article` (`id`),
  CONSTRAINT `comment_ibfk_2` FOREIGN KEY (`author_id`) REFERENCES `user` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `comment`
--

LOCK TABLES `comment` WRITE;
/*!40000 ALTER TABLE `comment` DISABLE KEYS */;
INSERT INTO `comment` VALUES (1,'KINKINKIN','2019-12-23 15:05:34',2,2);
/*!40000 ALTER TABLE `comment` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `user`
--

DROP TABLE IF EXISTS `user`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(64) DEFAULT NULL,
  `user_name` varchar(64) DEFAULT NULL,
  `email` varchar(120) DEFAULT NULL,
  `hashed_password` varchar(128) DEFAULT NULL,
  `social_id` varchar(64) DEFAULT NULL,
  `nickname` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `social_id` (`social_id`),
  UNIQUE KEY `ix_user_email` (`email`),
  UNIQUE KEY `ix_user_user_id` (`user_id`),
  KEY `ix_user_user_name` (`user_name`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `user`
--

LOCK TABLES `user` WRITE;
/*!40000 ALTER TABLE `user` DISABLE KEYS */;
INSERT INTO `user` VALUES (1,NULL,'yoon4480','yoon4480@naver.com',NULL,'facebook$2625513690857886',NULL),(2,'admin','관리자','admin@admin.com','pbkdf2:sha256:150000$2aAglbfi$0aa20f730532e7f4860d106c9020b622a5af0f4da2f707b433c5914ad27fc394',NULL,NULL);
/*!40000 ALTER TABLE `user` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2019-12-23  6:16:41

```

### 커스텀 MySQL 이미지 빌드

`dump.sql` 파일과 같은 디렉토리에 `Dockerfile`도 생성한다.

```Dockerfile
FROM mysql:8.0 as builder

RUN ["sed", "-i", "s/exec \"$@\"/echo \"not running $@\"/", "/usr/local/bin/docker-entrypoint.sh"]

ENV MYSQL_ROOT_PASSWORD=root

COPY dump.sql /docker-entrypoint-initdb.d/

RUN ["usr/local/bin/docker-entrypoint.sh", "mysqld", "--datadir", "/initialized-db"]

FROM mysql:8.0

COPY --from=builder /initialized-db /var/lib/mysql
```

MySQL의 기본 이미지를 사용하여 `builder`라는 alias를 지정해주고 덤핑한 데이터를 MySQL 컨테이너 내의 `/docker-endpoint-initdb.d/` 디렉토리에 복사한다.

`RUN ["usr/local/bin/docker-entrypoint.sh", "mysqld", "--datadir", "/initialized-db"]`: MySQL 기기본 이미지에 덤핑한 데이터를 복사하여 docker 빌드 프로세스를 수행하고, 기본 이미지에서 `/var/lib/mysql` 디렉토리로 `initialized-db`를 복사한다.

마지막으로 아래 커맨드로 MySQL 커스텀 이미지를 빌드한다.

```
$ docker build -t mysql-db-custom .
```

빌드가 잘 되나 싶더니만 에러를 만나버렸다. 흠.

> _2019-12-23 06:44:55+00:00 [Note] [Entrypoint]: /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/dump.sql_
>
> _ERROR 1046 (3D000) at line 22: No database selected_
>
> _The command 'usr/local/bin/docker-entrypoint.sh mysqld --datadir /initialized-db' returned a non-zero code: 1_

`dump.sql` 파일을 import하면서 데이터베이스가 선택되지 않았다는 내용 같은데 분명 덤핑할 때 명시했었다.
와 1시간동안 삽질한 결과 알아냈다. 일단 저 에러 메시지가 맞는 말을 던지기는 했다.

결론적으로, 우리가 MySQL 컨테이너를 띄울 때 `-e` 옵션으로 각 환경변수들을 설정하는 것처럼 이미지를 빌드할 때도 마찬가지로 환경변수들이 필요하다.

```
FROM mysql:8.0 as builder

RUN ["sed", "-i", "s/exec \"$@\"/echo \"not running $@\"/", "/usr/local/bin/docker-entrypoint.sh"]

ENV MYSQL_ROOT_PASSWORD=root
ENV MYSQL_USER=yoon
ENV MYSQL_PASSWORD=yoon
ENV MYSQL_DATABASE=flaboard

COPY dump.sql /docker-entrypoint-initdb.d/

RUN ["/usr/local/bin/docker-entrypoint.sh", "mysqld", "--datadir", "/initialized-db"]

FROM mysql:8.0

COPY --from=builder /initialized-db /var/lib/mysql
```

```
$ docker build -t mysql-db-custom .
```

```
...
2019-12-23 07:14:20+00:00 [Note] [Entrypoint]: Temporary server stopped

2019-12-23 07:14:20+00:00 [Note] [Entrypoint]: MySQL init process done. Ready for start up.

not running mysqld --datadir /initialized-db
Removing intermediate container 83bbdd3fd32e
 ---> 7c262d16c3dc
Step 9/10 : FROM mysql:8.0
 ---> d435eee2caa5
Step 10/10 : COPY --from=builder /initialized-db /var/lib/mysql
 ---> 4093f246116f
Successfully built 4093f246116f
Successfully tagged yoon4480/mysql-db-custom:latest
```

### 커스텀 MySQL 이미지 업로드

하, 감격의 순간이다. `docker push` 커맨드로 Docker Hub에 업로드하자.

```
$ docker push yoon4480/mysql-db-custom
```

```
The push refers to repository [docker.io/yoon4480/mysql-db-custom]
45faeb37d160: Pushed
55f5c7d40658: Pushed
8d0c9963a6ad: Pushed
17b62e7a629c: Pushed
8eae701cdfcf: Pushed
d4078c1b9fdb: Pushed
7055b7f82e4c: Pushed
2a9aab74013a: Pushed
414373ffccb4: Pushed
6599033b2ab2: Pushed
51734435c93c: Pushed
5a8a245abd1c: Pushed
99b5261d397c: Pushed
latest: digest: sha256:6dad54934989754f8840c49356f89a9bd30048c81b53033c495fc83c521d34c1 size: 3040
```

<img width="943" alt="image" src="https://user-images.githubusercontent.com/12066892/71342550-71a73500-25a0-11ea-8db0-faa638a8adac.png">
