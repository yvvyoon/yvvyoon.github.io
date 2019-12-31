---
title: '컨테이너 orchestration (2) Kubernetes'
date: 2019-12-30 18:48:00
categories: Kubernetes
---

이전 포스트에서 Docker의 빌트인 오케스트레이션 툴인 `Swarm`에 대해서 알아보았다. 이번 포스트에서는 구글 진영에서 나온 `Kubernetes(k8s)`에 대해 배운 내용을 개인적인 견해와 함께 정리해보고자 한다. 
회사에는 컨테이너를 비롯한 인프라를 운영하는 팀이 별도로 있기 때문에 필자가 직접 관리하는 업무를 맡을 가능성은 낮아보인다. 자세한 활용법보다는 전반적인 개념과 간단한 샘플 코드로 실습 정도만 해볼 예정이다.



## Kubernetes(k8s)가 뭐지?

컨테이너 환경에서 실무를 접하다보면 필요에 따라 혹은 의도치 않게 많은 양의 컨테이너를 구동하게 될 것이다. 필자가 개인적으로 공부하던 것처럼 로컬 서버 1대, 원격지 서버 1대 등의 환경에서만 구동하는 것이 아닌 무수히 많은 서버에서 구동하게 된다. 서버마다 하드웨어 스펙이 다를텐데 고사양 스펙의 서버에서 달랑 1~2개 컨테이너만 돌린다면 말 그대로 어마어마한 자원 낭비이다.   하드웨어 가용량 등의 상황을 고려하여 적절한 서버에 적절한 컨테이너를 배포할 필요가 생기는데 Kubernetes가 그러한 스케줄링, 모니터링 등의 관리 역할을 한다.

`GO`로 개발되었고 특정 벤더와 플랫폼에 종속되지 않아 퍼블릭 클라우드, 프라이빗 클라우드, 베어메탈 등의 거의 모든 운영 환경에서 사용이 가능하여 최근 각광을 받고 있다.

어느 분야든 마찬가지로 구조 파악이 무엇보다 중요하다. Kubernetes의 클러스터는 크게 `마스터`, `노드`로 구성된다. `마스터`는 속해 있는 클러스터의 전반적인 관리를 담당하고, `노드`는 컨테이너가 배포되는 머신이다. 마스터가 이 노드들을 관리하고, 1개 노드에는 1개 이상의 컨테이너들이 배포된다.



### Basic Object

Kubernetes에서 가장 기초가 되는 개념이다. `Pod`, `Volume`, `Service`, `Namespace`가 있다. `yaml` 또는 `json` 파일로 각 오브젝트의 스펙을 정의할 수 있다.

#### Pod

1개 이상의 컨테이너를 포함하는 단위이다. 1개 이상의 컨테이너를 포함하므로 컨테이너 단위로 배포하지 않고 Pod 단위로 배포한다. Pod 단위로 배포하는데에는 크게 2가지의 이유가 있다.

첫 번째로, 동일한 Pod 내에서 IP와 포트를 공유한다. 따라서 컨테이너 간에 IP로 통신하지 않고, `localhost:포트`로 통신할 수 있다. IP와 localhost가 뭐가 다르냐. IP는 랜카드를 통해 외부로 나갔다가 라우터를 거쳐서 되돌아오므로 방화벽의 영향을 받고, localhost는 IP 프로토콜의 `loopback`으로 접속하기 때문에 방화벽의 영향을 받지 않는다. 속도면에서도 차이가 있다.

두 번째로는, 컨테이너 간에 볼륨을 공유한다. 예를 들면, 일반적으로 애플리케이션과 로그 수집기는 별도의 컨테이너로 분리되어 배포되어 로그 수집기가 애플리케이션의 로그 내용을 읽는 것이 어려웠는데, Pod 내에서는 볼륨을 공유하므로 바로 애플리케이션의 로그 수집이 가능하다.

#### Volume

컨테이너 생성 시에 기본적으로 할당되는 스토리지는 컨테이너 중지 시 소멸되어 영구적이지 않다. 호스트의 디스크로 볼륨을 설정하여 아무리 컨테이너를 다운시켜도 데이터는 보존할 수 있다. Docker의 볼륨 개념과 그것을 나란히 한다.

다만 다른 것은 Kubernetes의 Pod 내에서는 이 볼륨을 컨테이너 간에 공유할 수 있다는 것이다.

`온프렘 기반 외장 스토리지(iSCSI, NFS)`, `클라우드 외장 스토리지(AWS EBS, Google PD)`, `오픈소스 기반(GitHub, Glusterfs)` 외장 스토리지 등의 다양한 스토리지들을 추상화하여 제공하고 있어 설계 시 다양한 선택지 마련이 가능하다.

#### Service

Pod는 1개 이상의 컨테이너를 포함하고, 서비스는 1개 이상의 Pod를 포함한다. 일반적으로 서비스에서 단일 Pod만으로 제공하는 경우는 드물다. 컨테이너를 재시작할 때 볼륨이 초기화되는 것처럼 IP 정보도 자동으로 바뀌게 되므로 로드 밸런서에서 Pod를 리스트업할 때 IP를 활용하는 것은 쉽지 않다. 로드 밸런서가 유연하게 Pod들을 관리하기 위해 `라벨(Label)`과 `라벨 셀렉터(Label selector)`라는 개념을 사용한다.

Pod 생성 시 `spec` 부분에 라벨 셀렉터를 지정하면 서비스가 라벨 셀렉터를 참조하여 동일한 라벨을 가진 Pod끼리 서비스를 묶는다.

- Equality based selector

    등식이 성립하는지 성립하지 않는지에 따라 리소스를 선택

    *environment = dev*

    *role != frontend*

- Set based selector

    집합의 개념을 사용하여 포함 여부에 따라 리소스를 선택

    *environment in (dev, prod, stage)*

    *role notin (frontend, backend, dba)*

```yaml
kind: Service
apiVersion: v1
metadata:
	name: my-service
spec:
	selector:
		app: myapp
	ports:
		- protocol: TCP
			port: 80
			targetPort: 9376
```

`app`이 `myapp`인 리소스들끼리 묶어 서비스를 생성한다.

#### Namespace

한 클러스터 내에서 Kubernetes 오브젝트들을 논리적으로 구분하는 단위이다. 네임스페이스 별로 분리하여 하드웨어의 할당량을 다르게 지정할 수도 있고, 네임스페이스에 따라 사용자의 접근 권한도 부여할 수 있다.



### Controller

Kubernetes의 기본 오브젝트들을 관리하는 역할을 하는 녀석이다. 

#### Replica Controller(RC)



> *replica: (실물을 모방하여 만든) 복제품, 모형*



- Pod Selector: 라벨에 따라 Pod를 구분하는 셀렉터이다.
- Replica 수: 지정한 Replica 수와 일치하도록 부족할 경우 추가, 초과화는 경우 삭제한다.
- Pod Template: Replica 수에 따라 자동으로 Pod를 생성할 때 템플릿 기초 정보들을 정의한다.

#### Replication Set

RC와 크게 다르지 않은 역할을 수행하고, 셀렉터에 따라 리소스를 선택하는 방식에 차이가 있다. RC가 Equality based selector인 반면, Replication Set은 이름에서 알 수 있듯이 Set based selector를 사용한다.

#### Deployment

다수의 배포 방법 중 일반적으로 많이 사용되는 `블루/그린` 배포 방식과 `롤링 업그레이드` 배포 방식이 있는데 배포 과정 중에 모니터링이 필요하고 유사 시 롤백해야 할 상황이 발생하게 된다. 배포 진행 중에 네트워크 연결이 끊어지는 경우 배포 자체도 끊어지는  상황을 초래할 수 있기 때문에 이런 배포 과정을 자동화하는 개념이다.

Pod 배포를 위한 RC 생성 및 관리, 롤백을 위한 기존 버전의 RC 관리 등 다양한 기능을 포함한다.



> *\<참고>*
>
> ​	*https://bcho.tistory.com/1256?category=731548*


