> 참고한 사이트는 다음과 같습니다.
> 
- [Docker 공식 문서](https://docs.docker.com/engine/swarm/swarm-tutorial/inspect-service/)
- [Docker Swarm의 주요 용어, 활성화 방법 및 노드(Node) 관리법 살펴보기](https://seongjin.me/docker-swarm-introduction-nodes/)
- [[Docker] Docker Swarm Cluster 구축하기(Swarm Mode)](https://yoo11052.tistory.com/183)
- [Docker Swarm에서 서비스(Service) 생성하고 다루기](https://seongjin.me/docker-swarm-services/)

> EC2 3개를 사용하여 Docker swarm 실습을 진행했습니다.
> 
- worker host: docker-swarm-2, docker-swarm-3
- manager host: docker-swarm-1

## 주요 용어

| 이름 | 설명 |
| --- | --- |
| 노드(Node) | 클러스터를 구성하는 개별 도커 서버를 의미한다. |
| 매니저 노드(Manager Node) | • 클러스터 관리와 컨테이너 오케스트레이션을 담당한다.
◦ 쿠버네티스의 마스터 노드와 같은 역할이라고 할 수 있다. |
| 워커 노드(Worker Node) | • 컨테이너 기반 서비스들이 실제 구동되는 노드를 의미한다.
◦ 쿠버네티스와 다른 점이 있다면, Docker swarm에서는 매니저 노드도 기본적으로 워커 노드의 역할을 같이 수행할 수 있다.
◦ 스케줄링을 임의로 막는 것도 가능하다. |
| 스택(Stack) | • 하나 이상의 서비스(Service)로 구성된 다중 컨테이너 애플리케이션 묶음을 의미한다.
◦ 도커 컴포즈와 유사한 양식의 yaml 파일로 스택 배포를 진행한다. |
| 서비스(Service) | • 노드에서 수행하고자 하는 작업들을 정의해놓은 것으로, 클러스터 안에서 구동시킬 컨테이너 묶음을 정의한 객체이다.
◦ 도커 스웜에서의 기본적인 배포 단위로 취급된다.
◦ 하나의 서비스는 하나의 이미지를 기반으로 구동되며, 이들 각각이 전체 애플리케이션의 구동에 필요한 개별적인 마이크로서비스(microservice)로 기능한다. |
| 테스크(Task) | • 클러스터를 통해 서비스를 구동시킬 때, 도커 스웜은 해당 서비스의 요구 사항에 맞춰 실제 마이크로서비스가 동작할 도커 컨테이너를 구성하여 노드에 분배하는데, 이를 테스크라고 한다.
◦ 하나의 서비스는 지정된 복제본(replica) 수에 따라 여러 개의 테스크를 가질 수 있으며, 각각의 테스크에는 하나씩의 컨테이너가 포함된다. |
| 스케줄링(Scheduling) | • 서비스 명세에 따라 테스크를 노드에 분배하는 작업을 의미한다.
◦ 2022년 8월 기준으로 도커 스웜에서는 오직 균등 분배(spread) 방식만 지원하고 있다. |

## 사용 포트

```bash
# AWS 인바웃드 규칙도 편집해야한다.
sudo ufw allow {port}
```

- `2377/tcp`: 클러스터 관리에 사용되는 포트
- `7946/tcp`, `7946/udp`: 노드 간 통신에 사용되는 포트
- `4789/udp`: 클러스터에서 사용되는 Ingress 오버레이 네트워크 트래픽에 사용된다.

## 사용 방법

### 매니저 노드 생성하기

```bash
docker swarm init --advertise-addr { public_ip }
```

- 매니저 호스트에 접속해 위 명령어를 입력해야한다.

### 워커 노드 생성하기

```bash
docker swarm join --token { token } { public_ip }:2377
# This node joined a swarm as a worker.
```

- 워커 호스트에 접속해 위 명령어를 입력한다.

```bash
ubuntu@ip-:~$ docker node ls
ID                            HOSTNAME           STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
dueqzu3vaew408t8awg6xe854     ip-                Ready     Active                          24.0.5
gb2bdfnxbqhcrqm4gqgtyxlj2 *   ip-                Ready     Active         Leader           24.0.5
dp24k5afwv6nprh0fqrty6uda     ip-                Ready     Active                          24.0.5
```

- 매니저 호스트에 접속해 list를 확인하면 다음과 같이 워커 노드가 추가된 것을 확인할 수 있다.
- `docker node ls` 명령어는 **매니저 노드에서만 동작**한다.
    - `ID` 값 옆에 붙은 `*`은 조회 명령을 수행한 매니저 노드를 의미한다.
- `ID`는 클러스터 차원에서 각 노드에 부여된 고유 값이며, `HOSTNAME`은 해당 노드의 이름이다.
    - `HOSTNAME`은 중복이 가능하지만 `ID`는 허용되지 않는다.
    - 호스트가 클러스터를 떠났다(leave)가 다시 합류(join)하는 경우 이전과 다른 새로운 `ID` 값이 부여된다.
- `AVAILABILITY`의 종류는 다음과 같다.
    - `Active`: 정상적으로 태스크를 할당받을 수 있는 상태
    - `Pause`: 스케줄러가 새로운 테스크 할당을 하지 않지만, 테스크가 구동 상태를 그대로 유지한다.
    - `Drain`: 스케줄러가 새로운 테스크를 할당하지 않고, 해당 노드에서 돌아가던 테스크가 모두 종료되며, `Active` 상태인 다른 노드로 다시 스케줄링 된다.
- `MANAGER STATUS`의 종류는 다음과 같다.
    - `Leader`: 스웜 클러스터의 관리와 오케스트레이션을 관리하는 노드이다.
    - `Reachable`: 매니저 노드로서 다른 매니저 노드들과 정상적으로 통신 가능한 상태이다.
        - 만약 `Leader` 노드에 장애가 발생하면, 해당 상태값을 가진 매니저 노드가 새 `Leader` 후보군이 된다.
    - `Unavailable`: `Leader`를 포함한 다른 매니저 노드들과 통신이 불가능한 상태이다.
- 클러스터는 1개 이상의 매니저 노드를 가지고 있고, `**Leader`로 선별된 매니저 노드가 전체 클러스터를 실질적으로 관리한다.**
    - 클러스터의 모든 변경 사항은 리더 노드를 통해 전파되고, 나머지 노드들이 리더 노드와 동기화된 상태를 유지한다.

## Service와 Task

### 서비스(Service)

- `docker run`은 컨테이너를 새로 생성하고 구동시키는 명령어이다.
    - 컨테이너 이름, 이미지 이름을 지정하는 것만으로도 도커 엔진이 알아서 필요한 이미지를 내려받아 구동시킨다.
- Docker의 기본 배포 단위가 컨테이너라면, **Docker Swarm의 기본 배포 단위는 서비스(Service)**이다.
    - 흔히 ‘같은 이미지에서 생성된 컨테이너의 집합’으로 알려져 있다.
    - 단일 이미지 기반으로 클러스터 안에서 구동시킬 컨테이너 묶음을 정의한 객체에 더 가깝다.
- 서비스는 선언적으로 구성딘다.
    - 즉, 원하는 상태를 정의하면, 도커는 그 상태를 유지하는데 필요한 작업을 지속적으로 수행한다.
    - 정의할 수 있는 요소들은 다음과 같다.
        1. 컨테이너로 배포할 이미지의 정보
        2. 전체 서비스를 구성할 컨테이너의 수
        3. 컨테이너의 배치 형태
        4. 컨테이너에 붙일 볼륨 정보
        5. 컨테이너를 배치시킬 노드 조건
        6. 컨테이너의 업데이트 전략 및 정책 지정

### 서비스(service)와 테스크(Task)의 관계

![Untitled](https://seongjin.me/content/images/2022/08/service-task.png)

- **선언적으로 정의된 서비스의 명세에 따라 생성되어 클러스터 노드에 배치되는 개별 컨테이너의 배포 단위가 바로 테스크(Task)이다.**
- 각 테스크에는 `서비스명.일련번호` 형태의 이름이 붙게 되며, 미리 정의된 컨테이너의 수(replicas) 만큼 생성되어 가용한 노드에 제각각 배치된다.
- 정의된 서비스의 명세를 스웜 매니저가 파악한 뒤, 가용한 노드에 지정된 레플리카 수 만큼의 테스크를 만들어 할당한다.

## Swarm에 서비스 배포

```bash
docker service create --replicas 1 --name helloworld alpine ping docker.com
```

- `ping docker.com` 명령어를 수행하는 컨테이너 서비스를 생성한다.
- 서비스 생성은 매니저 호스트에서만 사용할 수 있다.
- 옵션은 다음과 같다.
    - `--name`: 서비스의 이름을 지정한다.
    - `--replicas`: 서비스를 구성할 테스크(컨테이너)의 수를 지정한다. **지정된 숫자 만큼 테스크가 클러스터 노드에 배치된다.**
    - `alpine ping docker.com`: 테스크에 포함시킬 이미지 정보를 지정한다.

```bash
ubuntu@ip-:~$ docker service create --replicas 1 --name hello alpine ping docker.com
zpume3y3ng7rn67simnlvbwz5
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 

ubuntu@ip-:~$ docker service ls
ID             NAME          MODE         REPLICAS   IMAGE            PORTS
zpume3y3ng7r   hello         replicated   1/1        alpine:latest    

```

- `docker service ls`를 입력하면 생성된 서비스 목록을 볼 수 있다.

## Swarm에서 서비스 검사하기

```bash
ubuntu@ip-:~$ docker service inspect --pretty helloworld

ID:             m6n03eshissl401jy1ixc4j5i
Name:           helloworld
Service Mode:   Replicated
 Replicas:      1
Placement:
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         alpine:latest@sha256:82d1e9d7ed48a7523bdebc18cf6290bdb97b82302a8a9c27d4fe885949ea94d1
 Args:          ping docker.com 
 Init:          false
Resources:
Endpoint Mode:  vip

ubuntu@ip-:~$ docker service inspect helloworld
[
    {
        "ID": "m6n03eshissl401jy1ixc4j5i",
        "Version": {
            "Index": 21
        },
        "CreatedAt": "2023-08-02T11:35:37.50905067Z",
        "UpdatedAt": "2023-08-02T11:35:37.50905067Z",
        "Spec": {
            "Name": "helloworld",
            "Labels": {},
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "alpine:latest@sha256:82d1e9d7ed48a7523bdebc18cf6290bdb97b82302a8a9c27d4fe885949ea94d1",
                    "Args": [
                        "ping",
                        "docker.com"
                    ],
                    "Init": false,
                    "StopGracePeriod": 10000000000,
                    "DNSConfig": {},
                    "Isolation": "default"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "Delay": 5000000000,
                    "MaxAttempts": 0
                },
                "Placement": {
                    "Platforms": [
                        {
                            "Architecture": "amd64",
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "Architecture": "arm64",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "386",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "ppc64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "s390x",
                            "OS": "linux"
                        }
                    ]
                },
                "ForceUpdate": 0,
                "Runtime": "container"
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 1
                }
            },
            "UpdateConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "RollbackConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "EndpointSpec": {
                "Mode": "vip"
            }
        },
        "Endpoint": {
            "Spec": {}
        }
    }
]
```

- 서비스의 세부 정보를 볼 수 있다.
- `--pretty` 옵션을 제거하면 json 형식으로 반환한다.

```bash
ubuntu@ip-:~$ docker service ps helloworld
ID             NAME           IMAGE           NODE              DESIRED STATE   CURRENT STATE            ERROR     PORTS
dgeqqpgl329w   helloworld.1   alpine:latest   ip-               Running         Running 11 minutes ago
```

- 서비스를 실행중인 노드를 확인할 수 있다.
- 기본적으로 Swarm 관리자 노드는 워커 노드처럼 작업을 실행할 수 있다.

```bash
Error response from daemon: This node is not a swarm manager. 
Worker nodes can't be used to view or modify cluster state. 
Please run this command on a manager node or promote the current node to a manager.
```

- 만약 워커 호스트에서 서비스를 생성하면 다음과 같은 오류가 발생한다.

```bash
ubuntu@ip-:~$ docker ps
CONTAINER ID   IMAGE           COMMAND             CREATED         STATUS         PORTS     NAMES
6bd3b81b0e9d   alpine:latest   "ping docker.com"   4 minutes ago   Up 4 minutes             helloworld.1.a4vzxy4vqouxp6km0kqwbnc68
```

- 해당 작업을 실행중인 노드에서 `docker ps`를 실행하면 해당 작업의 세부 정보를 확인할 수 있다.

## Swarm에서 서비스 확장하기

```bash
ubuntu@ip-:~$ docker service scale hello=3
helloworld scaled to 3
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged
```

- `docker service scale` 명령어를 사용하면 배포된 서비스의 레플리카 수를 조정할 수 있다.

```bash
ubuntu@ip-:~$ docker service ps helloworld
ID             NAME               IMAGE           NODE              DESIRED STATE   CURRENT STATE            ERROR                              PORTS
a4vzxy4vqoux   helloworld.1       alpine:latest   ip-               Running         Running 2 hours ago                                         
dgeqqpgl329w    \_ helloworld.1   alpine:latest   ip-               Shutdown        Failed 2 hours ago       "No such container: helloworld…"   
n4b623sieipy   helloworld.2       alpine:latest   ip-               Running         Running 38 seconds ago                                      
dv6kdbneyykd   helloworld.3       alpine:latest   ip-               Running         Running 38 seconds ago
```

- `docker service ps ${service_name}`를 입력하면 업데이트된 작업 목록을 볼 수 있다.

## 발생했던 오류

```bash
Error response from daemon: Timeout was reached before node joined. 
The attempt to join the swarm will continue in the background. 
Use the "docker info" command to see the current swarm status of your node.
```

- private가 아닌 public ip로 매니저 노드를 지정해주면 된다.
- 우분투 내부에서 port 허용, AWS 자체에서 포트를 허용해준다.

```bash
ubuntu@ip-:~$ docker service ps helloworld
ID             NAME               IMAGE           NODE              DESIRED STATE   CURRENT STATE            ERROR                              PORTS
a4vzxy4vqoux   helloworld.1       alpine:latest   ip-               Running         Running 2 hours ago                                         
dgeqqpgl329w    \_ helloworld.1   alpine:latest   ip-               Shutdown        Failed 2 hours ago       "No such container: helloworld…"   
n4b623sieipy   helloworld.2       alpine:latest   ip-               Running         Running 38 seconds ago                                      
dv6kdbneyykd   helloworld.3       alpine:latest   ip-               Running         Running 38 seconds ago
```

- 왜 다른 노드로 클러스터 되지 않은 원인은 워커 노드가 다운되었기 때문..
    
    ```bash
    ubuntu@ip-:~$ docker node ls
    ID                            HOSTNAME           STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
    dueqzu3vaew408t8awg6xe854     ip-                Down      Active                          24.0.5
    gb2bdfnxbqhcrqm4gqgtyxlj2 *   ip-                Ready     Active         Leader           24.0.5
    dp24k5afwv6nprh0fqrty6uda     ip-                Down      Active                          24.0.5
    ```