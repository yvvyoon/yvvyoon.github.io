---
title: 'Docker 컨테이너 orchestration (1) Swarm'
date: 2019-12-27 18:48:00
categories: Docker
---

컨테이너 단위로 각 애플리케이션을 dockerize하여 개발, 배포, 운영 등의 과정을 간단하게 풀어나갈 수 있는 환경에 최근 IT 동향에서 큰큰 화제이다. 기존 개발 환경과는 굉장히 많은 차이점이 드러나는데, 대표적인 가상화 기술 중 하나인 `VM(Virtual Machine)`과 가상화라는 기본 컨셉을 나란히 하고 있지만 각 서비스들을 관리하는 측면에서 큰 차이점을 보이고 있다. Docker와 달리 VM 환경에서는 하나의 호스트 위에 1개 이상의 게스트 OS를 개별적으로 설치하여 동시에 사용할 수 있다. 그 이면에는 해당 OS를 사용하기 위해 필요한 기능, 필요하지 않은 기능들을 모두 설치하기 때문에 시간적, 자원적으로도 비용이 꽤 든다는 점이 있다. Docker는 호스트 OS의 Kernel 레이어를 공유하여 각 컨테이너들의 OS가 호스트 OS에 종속된다는 점이 있지만 필요한 서비스 부분만 설치하여 경량성(Lightweight)을 나타내기 때문에 그 궤를 달리한다.  
이전에 Flask 컨테이너와 MySQL 컨테이너를 `Link`를 통해 하나의 컨테이너처럼 결합했지만 동일한 Docker 호스트 안에서만 결합 및 통신이 가능했기 때문에 그 한계점이 뚜렷하다. 실제로 Link가 `deprecated`되었다는 얘기도 있어서 찾아보니 아직 명확한 내용은 찾을 수 없었다. `--link` 옵션은 `deprecated`되었고 `docker-compose.yml` 파일 내의 `link` 속성은 아직 남아 있다는 말도 있다.  
이와 같은 컨테이너 orchestration을 사용하려는 궁극적인 목적은 떨어져 있는 Docker 호스트에서 구동되고 있는 컨테이너들을 연결하려는데에 있다. 대표적인 orchestration 툴로는 Docker의 빌트인 기능인 `Swarm`, 구글에서 개발하여 최근에 굉장히 각광받고 있고 Docker 측에서도 공식적으로 채택하여 가이드 문서까지 제공하고 있는 `Kubernetes`, 빅데이터 환경의 대규모 클러스터링을 제공하는 아파치 재단의 `Mesos` 등이 있다.   
Swarm과 Kubernetes에 대해서만 간단히 학습해볼 것이고, Docker 빌트인 툴인 Swarm부터 시작해보자. 각 공식 문서를 바탕으로 개인적인 견해를 섞어서 작성하도록 하겠다.





## Swarm

`Swarm`은 `Smartkit`이라는 Docker orchestration을 담당하는 별도의 프로젝트를 통해 빌드되어 Docker 엔진에 내장된 클러스터 관리 및 orchestration 툴이다. `manager` 노드와 `worker` 노드 역할을 하는 다중 Docker 호스트로 구성된다. 동일한 이미지로부터의 컨테이너 수, 네트워크와 스토리지 리소스, 포트 등의 설정하여 최적의 서비스를 생성한다. 독립적인 컨테이너가 아닌 manager 노드가 관리하고 swarm 서비스의 일부분으로서의 컨테이너인 태스크의 스케줄링을 담당하기도 한다. 





### Swarm 특징

각 특징 제목에 대해 명확하게 번역할 말이 떠오르질 않아서 원문 그대로 사용했다. :). 

>  - **Cluster management integrated with Docker Engine: **Swarm을 사용하기 위한 별도의 orchestration 툴 필요없이 Docker 엔진의 CLI만으로도 생성 및 관리가 가능하다.
>
>  - **Decentralized design: ** 각 노드의 역할 분담을 배포 시점에 하지 않고 런타임에 한다. 단일 이미지만으로 전반적인 Swarm 클러스터를 구성할 수 있음을 의미한다.
>  - **Declarative service model: ** `YAML` 파일과 같은 선언적인 방법을 통해 하나의 애플리케이션 스택 안에 여러 서비스를 정의할 수 있다.
>  - **Scaling:** 각 서비스 안에서 원하는 갯수만큼의 태스크를 실행할 수 있다. manager 노드가 자동으로 태스크를 추가하고 삭제하면서 자동 스케일링을 수행한다.
>  - **Desired state reconciliation:** manager 노드가 지속적으로 모니터링하다가 서비스의 실제 상태와 요구된 상태가 일치하지 않을 때 그 간극을 좁혀준다. 10개의 컨테이너를 구동하도록 설계했는데 실제 운영 상황에서 2개 컨테이너가 다운된 경우, manager 노드가 자동으로 2개 컨테이너를 worker 노드들에게 할당해준다.
>  - **Multi-host networking:** 애플리케이션을 새로 시작하거나 업데이트할 때 각 컨테이너에 주소를 자동으로 할당하면서 오버레이 네트워크를 구성한다.
>  - **Service discovery:** manager 노드는 각 서비스에 고유의 DNS 이름을 할당하고 로드 밸런싱을 수행하는데, Swarm에 내장된 DNS 서버를 통해 실행되는 모든 컨테이너에 접근할 수 있다. 
>  - **Load balancing:** 외부 로드 밸런서에서 접근할 수 있도록 각 서비스의 포트를 노출시킨다. 내부적으로 노드 간 부하 분배 방식을 지정할 수 있다.
>  - **Secure by default:** 각 노드들은 안전한 통신을 위해 TLS 상호 인증 방식과 암호화 방식을 사용한다. Self-signed 인증서 또는 Root CA 인증서 중 선택해서 인증을 수행할 수 있다.
>  - **Rolling updates:** 서비스 출시 시점에 점진적으로 서비스를 업데이트할 수 있다. manager 노드가 서로 다른 노드 그룹들의 서비스 배포 간 딜레이를 컨트롤하는 역할을 한다. 비정상적인 상황이 발생하면 이전 버전으로 rollback할 수 있다.





### Swarm 핵심 개념

특징들로 보아 `docker-compose.yml` 파일로 여러 컨테이너들을 조합하여 마치 단일 컨테이너인 것처럼 배포하는 방식과 유사한 것 같다. Swarm의 4개의 핵심 개념을 살펴보자.





#### 노드

**노드(Node)**란 Swarm 클러스터에 참여한 Docker 엔진의 인스턴스를 말한다. Swarm 클러스터에 배포하기 위해 서비스에 대한 정의를 제출할 manager 노드와 manager 노드에 의해 각 태스크를 할당받는 worker 노드가 있다.





#### 서비스

Swarm에서 **서비스(Service)**는 각 노드가 수행할 태스크에 대한 정의를 뜻한다. 컨테이너의 기반 이미지를 정하거나 컨테이너 실행 시점의 커맨드를 설정하는 역할을 맡는다. `docker-compose.yml` 파일과 유사하다.





#### 태스크

**태스크(Task)**는 Swarm 클러스터 내에서의 가장 작은 작업 단위로서 위에서 언급했듯이 worker 노드가 manager 노드로부터 할당받는다. 한번 할당된 태스크는 다른 노드로 이동할 수 없고, 오로지 수행하거나 실패하거나로 나뉜다. 





#### 로드 밸런싱

Swarm 클러스터는 내부적으로 `Ingress 로드 밸런싱`을 사용하고 manager 노드가 `PublishedPort`를 서비스에 자동으로 할당해준다. 자동 할당을 원하지 않으면 사용하지 않을 포트를 포함하여 직접 설정해줄 수 있고, 포트를 설정하지 않으면 `30000`~`32767` 범위 내에서 manager 노드가 자동 할당한다. 특정 서비스가 구동 중인지, 중지되어 있는지에 상관없이 PublishedPort가 서비스에 명시되어 있으면 클라우드 로드 밸런서와 같은 외부 컴포넌트에서 접근이 가능하다.

내부에 DNS 컴포넌트도 가지고 있기 때문에 각 서비스에 DNS도 자동으로 할당해준다.





### CLI 커맨드

커맨드 아래에 달아놓은 설명이 무안할만큼 굉장히 직관적인 커맨드들을 제공하고 있다.





#### swarm init

Swarm 클러스터를 초기화한다. 이 커맨드가 수행되고 나면 `친절하게도` Swarm 클러스터에 참여할 때 사용할 토큰을 출력해준다.

```
$ docker swarm init [OPTIONS]
```





#### swarm join

manager 노드 또는 worker 노드로서 Swarm 클러스터에 참여하는 커맨드이다.

```
$ docker swarm join [OPTIONS] HOST:PORT
```





새로운 서비스를 생성하는 커맨드이다.

```
$ docker service create [OPTIONS] IMAGE [COMMAND] [ARG...]
```





#### service inspect

생성된 서비스들에 대한 상세한 설명을 출력해주는 커맨드이다.

```
$ docker service inspect [OPTIONS] SERVICE [SERVICE...]
```





#### service ls

생성된 서비스들의 목록을 출력한다.

```
$ docker service ls [OPTIONS]
```





#### service rm

생성된 서비스를 삭제한다.

```
$ docker service rm SERVICE [SERVICE...]
```





#### service scale

특정 서비스에 대해 원하는 수만큼 복제 서비스를 확장하며 축소시킬 수도 있다.

```
$ docker service scale SERVICE=REPLICAS [SERVICE=REPLICAS...]
```





#### service ps

`service ls`와 다르게 태스크들의 목록을 출력한다.

```
$ docker service ps [OPTIONS] SERVICE [SERVICE...]
```





#### service update

생성된 서비스에 변경이 필요할 경우 실행하는 커맨드이다. `service create` 커맨드에 사용되는 파라미터와 동일한 파라미터를 사용한다.

```
$ docker service update [OPTIONS] SERVICE
```





### Swarm 구동 확인

`docker system info` 명령어로 현재 Docker의 상태를 조회해볼 수 있고, `Swarm: active`인지 확인한다. `inactive` 상태라면 `docker swarm init` 명령어로 입력하면 된다.

```
Swarm initialized: current node (yfz977idzfh9gp76z5v0c79mp) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2wf73vmugm4thy4gbv1zaedsuvk1jsftr5audcdy22ju8puc2r-6nyhl6ll7zizxmmjrpzkcrr94 192.168.65.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

아 예전 기억이 조금씩 되살아나고 있다. `init` 명령어로 Swarm을 초기화하는 클러스터 관리자 역할의 `manager`, 클러스터에 참여하여 manager에 의해 컨테이너를 생성하고 상태를 체크하는 `worker`로 역할이 나뉜다. worker들은 발급된 토큰을 통해 네트워크에 참여할 수 있다.

컨테이너들을 연결하기 위해 `YAML` 스택 파일을 하나 만들어보자.





```yaml
version: '3.7'

services:
  flaboard:
    image: yoon4480/flaboard:0.2.6
      ports:
        - "5000:5000"
```





`services`라는 단위의 객체에 조합할 각 컨테이너들을 명시해준다. 컨테이너의 이름, 사용할 이미지, 포트 등을 설정한다.





> *\<참고>*
>
> ​	*https://docs.docker.com/get-started/part4/*
>
> ​	*https://docs.docker.com/engine/swarm/*
>
> ​	*https://subicura.com/2017/02/25/container-orchestration-with-docker-swarm.html*