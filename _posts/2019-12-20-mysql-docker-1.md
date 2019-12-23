---
title: 'Dockerize MySQL (1)'
date: 2019-12-20 18:00:00 -0400
categories: Docker
---

## Dockerize MySQL
<br>

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

`bridge` 네트워크 상에서 구동되고 있는 모든 컨테이너들의 네트워크 정보를 조회하는 커맨드이다.

```
$ docker network inspect bridge -f "{{json .Containers}}" | json_pp
```
```json
{
   "b60f020034cbb7675e4c28007f5722a6389c7fe6c4fac1a3555be02b812d3f81" : {
      "Name" : "mysql-db",
      "MacAddress" : "02:42:ac:11:00:02",
      "EndpointID" : "6ee2ffd48893a047c9f68f962e1c7792b85eb3d364b7045c828edd72be121533",
      "IPv6Address" : "",
      "IPv4Address" : "172.17.0.2/16"
   }
}
```

위의 IP로 MySQL 컨테이너에 접속을 테스트해봤는데 실패했다. Flask에서 migrate도 먹히질 않는다. 왜지. 방화벽인가. 인바운드 때문인가. 컨테이너를 생성할 때마다 초기 CIDR 설정에 의해 자동으로 할당되는 IP 때문인 것 같은데 정확한 원인은 알아봐야겠다.

DB 접속 정보를 `localhost`에서 `127.0.0.1`로 바꿔서 IP 주소를 명시해줬더니 신기하게도 접속이 된다. `localhost`로 접속했하는 경우 MySQL이 포트 번호를 무시해버린다. 관련 내용을 좀 알아보았다.

> _On Unix, MySQL programs treat the host name localhost specially, in a way that is likely different from what you expect compared to other network-based programs. For connections to localhost, MySQL programs attempt to connect to the local server by using a Unix socket file. This occurs even if a --port or -P option is given to specify a port number. To ensure that the client makes a TCP/IP connection to the local server, use --host or -h to specify a host name value of 127.0.0.1, or the IP address or name of the local server. You can also specify the connection protocol explicitly, even for localhost, by using the --protocol=TCP option._

위 내용은 MySQL 공식 문서 내용 일부를 발췌한 것이다.

Unix에서의 MySQL은 `localhost`를 우리가 일반적으로 알고 있는 것과 다른 방식으로 접속한다. `TCP/IP` 방식으로 접속하지 않고 `소켓` 방식으로 연결하는데 `--port`, `-p` 옵션으로 포트 번호를 지정해도 `localhost`라는 호스트 이름은 무시된다. TCP/IP 방식으로 연결하려면 `--host`, `-h` 옵션 뒤에 `127.0.0.1` 또는 IP 주소를 명시해줘야 한다. `localhost` 사용이 불가능한 것은 아니다. `--protocol=localhost`라고 명시해주면 되긴 하는데, 앞으로 혹시 모를 또다른 충돌을 미연에 방지하기 위해 `localhost` 대신 `127.0.0.1`로 설정하는 습관을 들여야겠다.
