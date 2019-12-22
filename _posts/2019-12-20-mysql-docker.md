---
title: 'Dockernize MySQL'
date: 2019-12-20 18:00:00 -0400
categories: Docker
---

## Dockernize MySQL

이전에 매번 MySQL 계정 및 데이터베이스 설정하는 게 귀찮아 Github에 올렸던 적이 있다. 이제 데이터 쌓는 것까지 너무 귀찮아서 데이터까지 통째로 포함하여 Docker 컨테이너를 pull/push하려고 한다.

일단 내 Docker Hub에 이미지 올렸던 경험은 있는데 데이터까지 말아서 올릴 수 있는지는 아직 확인해보지 않았다.

<br>

### MySQL 컨테이너 구동

```
$ docker pull mysql
$ docker run --name mysql-db -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql
```

<br>

`e` 옵션은 `Environment Variables`의 약어로서 총 4개의 변수를 포함한다.

<br>

> _MYSQL_ROOT_PASSWORD (필수)_
>
> _MYSQL_DATABASE: 데이터베이스 생성 시 사용_
>
> _MYSQL_USER, MYSQL_PASSWORD: 사용자를 생성하고 생성된 데이터베이스의 권한을 얻음_
>
> _MYSQL_ONETIME_PASSWORD: 1회성 패스워드 생성_

<br>

```
$ docker exec -it mysql-db bash
```

<br>

CLI로 들어왔으면 MySQL에 접속해서 데이터베이스와 사용자를 생성해준다.

여기서 잠깐! CLI로 매번 사용자 권한과 데이터베이스 생성이 귀찮으므로 위에서 언급했던 변수들을 사용하여 컨테이너를 띄울 것이다. 접속해서 CLI로 조회해보면 사용자와 데이터베이스가 잘 생성되어 있다.

```
$ docker run --name mysql-db -p 3306:3306 -d \
-e MYSQL_ROOT_PASSWORD=root \
-e MYSQL_USER=yoon \
-e MYSQL_PASSWORD=yoon \
-e MYSQL_DATABASE=flaboard \
-d mysql
```

<br>

### 외부에서의 컨테이너 접속

이제 내가 구축한 서버에서 이 MySQL 컨테이너로 접속하려면 이 컨테이너의 가상 IP를 알아내야 한다. 아래 커맨드는 `특정 컨테이너`의 네트워크 정보를 조회하는 것이다. `json.` 다음에 띄어쓰기 꼭 넣어! 객체 참조가 아니라 인자로 받는 것이다.

```
$ docker inspect mysql-db -f "{{json. NetworkSettings.Networks}}"
```

```json
{
	"bridge": {
		"IPAMConfig": null,
		"Links": null,
		"Aliases": null,
		"NetworkID": "8b21c5994c4eb65a3ec24af1d68c1c171553049741d1262e4a23b1de630782a1",
		"EndpointID": "adb3c07e49cd454c04de20de69a964af3f52350513ea544d4216894cce24ea2a",
		"Gateway": "172.17.0.1",
		"IPAddress": "172.17.0.2",
		"IPPrefixLen": 16,
		"IPv6Gateway": "",
		"GlobalIPv6Address": "",
		"GlobalIPv6PrefixLen": 0,
		"MacAddress": "02:42:ac:11:00:02",
		"DriverOpts": null
	}
}
```

예쁘지 않다. 아프다. 눈이.

<br>

```
$ docker inspect mysql-db -f "{{json. NetworkSettings.Networks}}" | json_pp
```

```json
{
	"bridge": {
		"Links": null,
		"IPAddress": "172.17.0.2",
		"Gateway": "172.17.0.1",
		"DriverOpts": null,
		"EndpointID": "adb3c07e49cd454c04de20de69a964af3f52350513ea544d4216894cce24ea2a",
		"IPAMConfig": null,
		"Aliases": null,
		"MacAddress": "02:42:ac:11:00:02",
		"IPv6Gateway": "",
		"IPPrefixLen": 16,
		"GlobalIPv6PrefixLen": 0,
		"GlobalIPv6Address": "",
		"NetworkID": "8b21c5994c4eb65a3ec24af1d68c1c171553049741d1262e4a23b1de630782a1"
	}
}
```

<br>

위의 IP로 MySQL 컨테이너에 접속을 테스트해봤는데 실패했다. Flask에서 migrate도 먹히질 않는다. 왜지. 방화벽인가. 인바운드 때문인가.

<br>

```
$ docker network inspect bridge -f "{{json .Containers}}" | json_pp
```

<br>

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

<br>

`volumes`에 생성할 볼륨의 이름을 정하고 원하는 경로를 설정한다. `${volume_name}:${volume_path}` 형식으로 지정하면 된다.

docker-compose.yml에 설정해보기 전에 `-v` 옵션으로 볼륨을 지정해보자. OS 별로 경로가 다르기 때문에 주의하자. 아래 커맨드는 macOS 기준이다. 일반적으로 리눅스 환경에서는 `/var/lib/docker/` 경로로 지정하긴 하지만 결국 개인의 자유다.

필자는 별도의 디렉토리를 생성해서 설정할 것이다.

<br>

> _참고:_
>
> ​ _https://medium.com/@crmcmullen/how-to-run-mysql-in-a-docker-container-on-macos-with-persistent-local-data-58b89aec496a_

<br>

```
docker run --name mysql-db -p 3306:3306 -d \
-e MYSQL_ROOT_PASSWORD=root \
-e MYSQL_USER=yoon \
-e MYSQL_PASSWORD=yoon \
-e MYSQL_DATABASE=flaboard \
-d mysql -v /Users/user/workspace/mysql-data:/var/lib/mysql
```

```
e265d1e7b550e160bde3133d65c06dafbe09e3b19fdad351ec2a93dc8836831f
```
