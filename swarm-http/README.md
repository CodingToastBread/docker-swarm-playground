# swarm-http: 맥북 한 대로 Docker Swarm HTTP 서비스 실습하기 (DinD + HAProxy)

맥북(Docker Desktop) 한 대에서 **Docker-in-Docker(DinD)** 컨테이너 3개로 Docker Swarm 클러스터(manager 1 + worker 2)를 만들고, 서비스 배포와 로드밸런싱, 노드 장애를 실습하는 환경입니다.

실습하면서 두 가지 문제를 만났고, 원인을 추적해 해결했습니다.

1. **`curl localhost:8080`이 3번 중 1번만 성공**
   - 원인: Docker Desktop이 넘겨준 패킷의 TCP 체크섬이 비어 있어, swarm이 다른 노드로 VXLAN 전달하면 수신 노드가 버렸습니다.
   - 해결: 앞단에 프록시(HAProxy)를 둬서 TCP 연결을 새로 맺게 했습니다.
2. **재시작 후 정상화까지 약 1분 45초**
   - 원인: gossip 접속 순서 경쟁과, dockerd가 일부러 넣는 15초 시작 지연이 겹쳤습니다.
   - 해결: healthcheck, `depends_on`, dockerd 시작 인자 조정으로 **약 10초**까지 줄였습니다.

또한 이후 CI/CD(레지스트리 push/pull, 원격 배포)로 확장할 것을 고려해, **manager의 Docker API를 TLS(사설 CA 인증서)로 열어** Mac이나 다른 컨테이너에서 원격으로 조작할 수 있게 했습니다.

> 이 문서의 수치와 로그는 모두 아래 환경에서 직접 측정한 값입니다. 직접 확인하지 못하고 추정한 부분은 **(추정)**이라고 표시했습니다.

---

## 목차

- [테스트 환경](#테스트-환경)
- [구조](#구조)
- [빠른 시작](#빠른-시작)
- [파일 구성 설명](#파일-구성-설명)
- [개념 정리: swarm ingress(routing mesh)](#개념-정리-swarm-ingressrouting-mesh)
- [트러블슈팅 1: curl이 3번 중 1번만 성공 (VXLAN 체크섬)](#트러블슈팅-1-curl이-3번-중-1번만-성공-vxlan-체크섬)
- [트러블슈팅 2: 재시작 후 정상화까지 너무 오래 걸림](#트러블슈팅-2-재시작-후-정상화까지-너무-오래-걸림)
- [노드 장애 실습](#노드-장애-실습)
- [manager 원격 접속 (TLS)](#manager-원격-접속-tls)
- [디버깅 명령 모음](#디버깅-명령-모음)
- [참고 자료](#참고-자료)

---

## 테스트 환경

| 항목 | 버전 |
|---|---|
| macOS | 27.0 (arm64) |
| Docker Desktop 엔진 | 29.8.0 |
| Docker Compose | v5.5.1 |
| 노드 이미지 | `docker:29-dind` (안쪽 엔진 29.8.1) |
| LB 이미지 | `haproxy:3.2-alpine` (3.2.24) |
| 서비스 이미지 | `traefik/whoami` (요청을 받은 컨테이너의 Hostname과 IP를 응답) |

---

## 구조

```
 Mac  ── localhost:8080 (서비스), localhost:8404 (HAProxy 통계)
  │      127.0.0.1:2376 ──────────────────────▶ manager Docker API (TLS, 아래 참고)
  │  Docker Desktop 포트포워딩
  ▼
 ┌──────────────┐              swarm-net: 172.30.0.0/24 (compose 브리지 네트워크)
 │ lb (HAProxy) │ 172.30.0.2
 └──────┬───────┘
        │  새 TCP 연결, 세 노드로 라운드로빈 + health check
        ├──────────────────┬──────────────────┐
        ▼                  ▼                  ▼
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ manager      │   │ worker1      │   │ worker2      │   ← 각각 docker:29-dind 컨테이너
 │ 172.30.0.10  │   │ 172.30.0.11  │   │ 172.30.0.12  │
 │  dockerd     │   │  dockerd     │   │  dockerd     │   ← 안쪽 dockerd끼리 swarm 구성
 │   └ web.x    │   │   └ web.x    │   │   └ web.x    │   ← 서비스 컨테이너는 노드 "안"에서 실행
 └──────────────┘   └──────────────┘   └──────────────┘
        ▲                  ▲                  ▲
        └───── swarm overlay (VXLAN, UDP 4789) ┘
```

```
swarm-http/
├── docker-compose.yml     # 노드 3개 + lb
├── haproxy/haproxy.cfg    # HAProxy 설정
├── stacks/whoami.yml      # (선택) docker stack deploy용 예제
├── certs/manager-client/  # manager Docker API용 클라이언트 인증서 (자동 생성, git 제외)
└── README.md
```

---

## 빠른 시작

이 문서의 명령은 모두 `swarm-http/` 디렉터리에서 실행합니다.

```bash
cd swarm-http
docker compose up -d
docker ps                                                # manager, worker1, worker2, lb 확인
docker exec manager docker info                          # 에러 없이 나오면 준비 완료
docker exec manager docker info --format '{{.Driver}}'   # overlayfs (아래 참고)

# manager에서 swarm 시작
docker exec manager docker swarm init --advertise-addr 172.30.0.10

# worker 참여용 토큰 확인 (출력값 복사)
docker exec manager docker swarm join-token -q worker

# worker 참여
docker exec worker1 docker swarm join --token <토큰> 172.30.0.10:2377
docker exec worker2 docker swarm join --token <토큰> 172.30.0.10:2377

# 확인: 3개 노드 모두 Ready
docker exec manager docker node ls

# 생성: whoami 컨테이너 3개, 8080 포트로 노출
docker exec manager docker service create --name web --replicas 3 -p 8080:80 traefik/whoami

# 목록, 배치, 상세
docker exec manager docker service ls
docker exec manager docker service ps web
docker exec manager docker service inspect --pretty web

# 접속 확인 (여러 번 실행하면 Hostname이 바뀜)
curl localhost:8080

# HAProxy 통계 페이지: 브라우저에서 http://localhost:8404
```

> **Driver가 `overlay2`가 아니라 `overlayfs`로 나옵니다**
> dockerd 로그에 `containerd-snapshotter=true storage-driver=overlayfs`라고 나옵니다. 이미지 저장을 containerd snapshotter가 맡는 구성이라 이렇게 표시되고, 정상입니다.

> **Hostname이 `A → B → C` 순서로 돌지 않고 섞여 나옵니다**
> HAProxy와 swarm이 각각 한 번씩, 두 단계로 분산하기 때문입니다. [개념 정리](#두-단계-분산-hostname-순서가-섞이는-이유)에서 자세히 설명합니다. 요청이 많아지면 고르게 나뉩니다(30회 요청 시 9/10/11회).

### 스택 배포 (선택)

`./stacks`는 manager의 `/stacks`에 마운트되어 있습니다.

```bash
docker exec manager docker stack deploy -c /stacks/whoami.yml demo
```

`stacks/whoami.yml`도 8080 포트를 쓰므로, 위의 `web` 서비스와 동시에 띄울 수 없습니다. 먼저 `docker exec manager docker service rm web`으로 지우세요.

### 재시작 / 초기화

```bash
docker compose down && docker compose up -d   # 볼륨 유지: swarm과 서비스가 그대로 살아남 (약 10초 후 정상 응답)
docker compose down -v                        # 볼륨까지 삭제: swarm을 처음부터 다시 구성
```

`down -v`를 하면 manager의 사설 CA도 새로 만들어집니다. `./certs/manager-client`는 다음 시작 때 새 CA 기준으로 자동 갱신되지만, 그 전에 다른 곳에 복사해 둔 클라이언트 인증서는 더 이상 쓸 수 없습니다. [manager 원격 접속](#인증서-유지-방식) 참고.

---

## 파일 구성 설명

### docker-compose.yml: 노드 공통 (`manager`, `worker1`, `worker2`)

| 항목 | 값 | 설명 / 필요한 이유 |
|---|---|---|
| `image` | `docker:29-dind` | 컨테이너 안에서 dockerd를 실행하는 공식 DinD 이미지입니다. 컨테이너 하나가 "Docker가 설치된 서버 한 대" 역할을 합니다. |
| `container_name` | `manager` 등 | `docker exec manager ...`처럼 고정된 이름으로 접근하려고 지정했습니다. |
| `hostname` | `manager` 등 | swarm은 노드 이름으로 hostname을 씁니다. 지정하지 않으면 `docker node ls`에 무작위 컨테이너 ID가 나옵니다. |
| `privileged` | `true` | 안쪽 dockerd가 네트워크 네임스페이스, iptables, VXLAN 인터페이스, cgroup, 파일시스템 마운트를 만들려면 커널 권한이 필요합니다. DinD의 필수 조건입니다. |
| `networks.swarm-net.ipv4_address` | `172.30.0.10/11/12` | IP를 고정합니다. `swarm init --advertise-addr`와 `swarm join`에 쓰는 주소이고, 재시작 후에도 같아야 볼륨에 저장된 swarm 상태가 그대로 유효합니다. |
| `volumes` | `<node>-data:/var/lib/docker` | 안쪽 dockerd의 데이터(이미지, 컨테이너, **swarm raft 상태**)를 named volume에 보관합니다. `compose down/up` 후에도 swarm과 서비스가 유지되고, 이미지를 다시 받지 않습니다. DinD에서는 `/var/lib/docker`를 컨테이너 자체 파일시스템이 아닌 볼륨에 두는 것이 일반적입니다. |

#### Docker API 접속 통로: unix 소켓과 TCP

`docker` CLI는 dockerd에 API로 명령을 보내는데, 접속 통로가 두 가지입니다.

| 통로 | 주소 | 접속 가능한 대상 |
|---|---|---|
| unix 소켓 | `/var/run/docker.sock` (파일) | 같은 머신(컨테이너) 안의 프로세스만 |
| TCP | `tcp://0.0.0.0:2375` (TLS 없음) / `:2376` (TLS) | 네트워크로 닿는 누구나 (TLS면 인증서를 가진 쪽만) |

`docker exec manager docker node ls`를 실행하면 **manager 컨테이너 안의** docker CLI가 기본값인 unix 소켓으로 안쪽 dockerd에 붙습니다. 그래서 `docker exec`로 조작할 때는 TCP API가 필요 없습니다. swarm 노드끼리도 API 포트가 아니라 2377, 7946, 4789를 씁니다.

이 실습에서는 노드별로 다르게 설정했습니다.

| 노드 | Docker API | 이유 |
|---|---|---|
| manager | unix 소켓 + **TCP 2376 (TLS)** | Mac이나 CI 컨테이너에서 원격으로 swarm을 조작(배포 등)하기 위해 |
| worker1, worker2 | unix 소켓만 | 원격으로 조작할 일이 없으므로 API 포트를 열지 않음 |

#### `command`와 `DOCKER_TLS_CERTDIR`의 관계

dind 이미지의 entrypoint(`dockerd-entrypoint.sh`)는 **인자가 없거나 `-`로 시작할 때만** 기본 인자를 붙입니다. `DOCKER_TLS_CERTDIR`는 이때 동작을 정합니다.

| `DOCKER_TLS_CERTDIR` | entrypoint가 붙이는 인자 |
|---|---|
| `/certs` (이미지 기본값) | 인증서 자동 생성 + `tcp://0.0.0.0:2376 --tlsverify` |
| `""` (빈 값) | `tcp://0.0.0.0:2375` (TLS 없음) → 15초 시작 지연 |

- **worker:** `command`를 `dockerd --host=unix:///var/run/docker.sock`로 직접 지정해서 이 분기를 건너뜁니다. `DOCKER_TLS_CERTDIR`는 아무 영향이 없어 설정하지 않았습니다. 확인 결과 dockerd 인자는 unix 소켓 하나뿐이고, 인증서는 생성되지 않았습니다(`/certs/client`는 이미지에 원래 있는 빈 디렉터리).
- **manager:** `command`를 지정하지 않고 `DOCKER_TLS_CERTDIR=/certs`를 명시해서, entrypoint가 인증서를 만들고 TLS로 2376을 열게 했습니다.
- `""`(TLS 없는 2375)는 어느 노드에도 쓰지 않습니다.

### docker-compose.yml: manager 전용

| 항목 | 설명 / 필요한 이유 |
|---|---|
| `command` 없음 + `environment.DOCKER_TLS_CERTDIR: /certs` | entrypoint 기본 동작으로 dockerd를 `--host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify ...`로 띄웁니다. 인증서는 `/certs`에 사설 CA로 자동 생성합니다. `/certs`는 이미지 기본값이지만 의도를 드러내려고 명시했습니다. |
| `ports: 127.0.0.1:2376:2376` | Docker API(TLS)를 Mac의 **127.0.0.1에만** 엽니다. 사내망 등 외부에서는 접근할 수 없습니다. |
| `volumes: manager-certs:/certs` | CA, 서버, 클라이언트의 **개인키**를 named volume에 보관합니다. 컨테이너를 새로 만들어도 같은 키를 쓰므로 기존 클라이언트 인증서가 계속 유효합니다. |
| `volumes: ./certs/manager-client:/certs/client` | 클라이언트 인증서(`ca.pem`, `cert.pem`, `key.pem`)만 호스트로 꺼냅니다. Mac이나 다른 컨테이너가 이 파일로 접속합니다. CA 개인키는 볼륨 안에만 있습니다. `.gitignore`로 커밋에서 제외합니다. |
| `volumes: ./stacks:/stacks` | 호스트의 스택 파일을 manager 안에서 `docker stack deploy -c /stacks/...`로 쓰기 위한 마운트입니다. |
| `healthcheck` | dockerd가 응답하고, swarm 상태라면 **gossip 포트 7946까지 열린 뒤에** healthy로 판정합니다. swarm을 아직 만들지 않은 처음 상태에서는 dockerd만 떠 있으면 통과합니다. |

```sh
docker info >/dev/null 2>&1 &&                                         # dockerd가 응답하는지
{ [ "$(docker info --format '{{.Swarm.LocalNodeState}}')" != active ]   # swarm 전이면 통과
  || nc -z 127.0.0.1 7946; }                                            # swarm 상태면 7946이 열려야 통과
```

compose 파일 안에서는 `$`를 `$$`로 써야 compose의 변수 치환을 피할 수 있습니다.

### docker-compose.yml: worker 전용

| 항목 | 설명 / 필요한 이유 |
|---|---|
| `command` | `["dockerd", "--host=unix:///var/run/docker.sock"]`로 **unix 소켓만** 엽니다. 이게 없고 `DOCKER_TLS_CERTDIR: ""`이던 초기에는 TLS 없는 2375가 자동으로 열리면서 dockerd 시작이 **약 15초 지연**됐습니다. [트러블슈팅 2](#원인-b-dockerd의-의도적인-15초-시작-지연-40초--10초) 참고. |
| `depends_on.manager.condition: service_healthy` | manager가 healthy가 된 뒤에 worker를 시작합니다. 재시작할 때 worker가 manager의 gossip보다 먼저 접속을 시도해 실패하는 경쟁 상태를 막습니다. [트러블슈팅 2](#원인-a-gossip-접속-순서-경쟁-1분-45초--40초) 참고. |

### docker-compose.yml: `lb`

| 항목 | 값 | 설명 / 필요한 이유 |
|---|---|---|
| `image` | `haproxy:3.2-alpine` | 외부 로드밸런서입니다. 맥에서 온 연결을 **여기서 끊고**, 노드 쪽으로 **새 TCP 연결**을 맺습니다. |
| `ports` | `8080:8080` | 맥의 `localhost:8080`을 lb로 연결합니다. **노드 컨테이너에 직접 포트를 매핑하지 않는 것**이 핵심입니다. [트러블슈팅 1](#트러블슈팅-1-curl이-3번-중-1번만-성공-vxlan-체크섬) 참고. |
| `ports` | `8404:8404` | HAProxy 통계 페이지입니다. 노드별 UP/DOWN과 요청 수를 볼 수 있습니다. |
| `volumes` | `./haproxy/haproxy.cfg:...:ro` | 설정 파일을 읽기 전용으로 마운트합니다. |
| `ipv4_address` | `172.30.0.2` | 노드 IP와 겹치지 않게 고정했습니다. |
| `depends_on` | `manager`, `worker1`, `worker2` | 노드가 뜬 뒤에 시작합니다. 먼저 떠도 health check가 노드를 DOWN으로 두었다가 살아나면 UP으로 바꾸므로 필수는 아닙니다. |

### docker-compose.yml: `networks` / `volumes`

| 항목 | 설명 |
|---|---|
| `networks.swarm-net` (subnet `172.30.0.0/24`) | 노드끼리 통신하는 "물리 네트워크" 역할의 브리지 네트워크입니다. 고정 IP를 쓰려면 subnet을 명시해야 합니다. |
| `volumes` | 노드별 `/var/lib/docker`용 volume과 manager의 인증서용 `manager-certs`입니다. `docker compose down -v`로 삭제됩니다. |

swarm이 노드끼리 쓰는 포트는 다음과 같습니다. 같은 compose 네트워크 안에서는 모두 열려 있어 별도 설정이 필요 없습니다.

| 포트 | 용도 |
|---|---|
| 2377/tcp | swarm 관리 (raft, join) |
| 7946/tcp+udp | gossip: "어느 노드에 어떤 task가 있는지"를 노드끼리 공유 |
| 4789/udp | VXLAN: overlay 네트워크의 실제 데이터 |

### haproxy/haproxy.cfg

```
frontend web          bind :8080  →  backend swarm_nodes
backend swarm_nodes   manager / worker1 / worker2 의 :8080
frontend stats        bind :8404  (통계 페이지)
```

| 설정 | 설명 / 필요한 이유 |
|---|---|
| `server ... 172.30.0.1x:8080` | 세 노드의 published port입니다. routing mesh 덕분에 **어느 노드로 보내도** 서비스가 응답하므로, HAProxy는 replica 위치를 몰라도 됩니다. |
| `balance roundrobin` | 세 노드에 번갈아 보냅니다. |
| `option httpchk GET /` + `check inter 2s fall 2 rise 2` | 2초마다 각 노드에 HTTP 요청을 보내 상태를 확인합니다. 2번 연속 실패하면 DOWN(제외), 2번 연속 성공하면 UP(복귀)입니다. 포트만 확인하는 TCP 체크와 달리 **실제 응답까지** 확인합니다. |
| `retries 2` + `option redispatch` | 연결에 실패하면 **다른 노드로** 재시도합니다. health check가 장애를 알아채기 전의 요청을 살리는 용도입니다. |
| `timeout connect 2s` | 죽은 노드로의 연결을 빨리 포기하고 재시도로 넘어가게 합니다. |
| `http-reuse never` | 백엔드 연결을 요청 간에 재사용하지 않습니다. 재사용하면 연결 하나가 노드 IPVS의 특정 replica에 고정되어 분산이 쏠릴 수 있습니다. |
| `frontend stats` | 통계 페이지입니다. 2초마다 자동으로 새로고침됩니다. |

---

## 개념 정리: swarm ingress(routing mesh)

트러블슈팅을 이해하려면 swarm이 published port(`-p 8080:80`)를 어떻게 처리하는지 알아야 합니다.

### routing mesh의 동작

`-p 8080:80`으로 서비스를 만들면 **모든 노드**가 8080을 엽니다. replica가 없는 노드도 마찬가지입니다. 노드의 8080 뒤에는 `ingress_sbox`라는 네트워크 네임스페이스 안의 **IPVS 로드밸런서**가 있습니다. 이 IPVS는 gossip으로 받은 **클러스터 전체 replica 목록**을 갖고 있습니다.

```
worker1 노드
┌───────────────────────────────────────────────────────────┐
│  :8080 ──▶ ingress_sbox (IPVS)                            │
│              "web 서비스의 replica 목록"                   │
│               ├─ manager에 있는 replica ──VXLAN──▶ 다른 노드로
│               ├─ worker1에 있는 replica ──▶ 로컬 컨테이너  │
│               └─ worker2에 있는 replica ──VXLAN──▶ 다른 노드로
└───────────────────────────────────────────────────────────┘
```

IPVS는 "내 노드의 replica를 우선"하지 않고, 목록에서 순서대로 고릅니다. 그래서 **요청의 상당수가 다른 노드로 넘어갑니다.** replica가 노드마다 하나씩이면 3분의 2가 넘어갑니다.

**manager가 특별한 게 아닙니다.** 요청을 나눠 주는 일은 모든 노드가 똑같이 합니다. manager의 역할은 관리(서비스 생성, replica 배치, 장애 시 재생성)입니다.

### 왜 이렇게 설계했나: 효율 대신 유연성

노드 간 이동은 비효율이지만, 그 대가로 **외부 LB가 replica 위치를 몰라도 됩니다.**

```
내주는 것                          얻는 것
──────────────────────            ────────────────────────────────────
노드 간 이동 (VXLAN 한 번 더)        replica가 옮겨가거나 개수가 바뀌어도 외부 LB 설정 그대로
                                  replica 없는 노드로 들어와도 응답함
```

Docker 공식 문서의 [Use swarm mode routing mesh](https://docs.docker.com/engine/swarm/ingress/) 페이지에도 HAProxy를 앞에 두고 세 노드의 8080으로 분산하는 예시가 있습니다. 이 실습의 구성과 같은 구조입니다.

### 실험: replica가 없는 노드도 응답할까

replica를 2개로 줄여 manager에는 replica가 없게 만들었습니다.

```bash
docker exec manager docker service scale web=2
```

```
manager  : replica 없음
worker1  : web.1 (10.0.0.10)
worker2  : web.2 (10.0.0.9)
```

HAProxy를 거치지 않고 **manager:8080으로 직접** 6회 요청했습니다(lb 컨테이너에서 `wget http://172.30.0.10:8080`).

```
Hostname: ad9f8613a454   (worker1)
Hostname: 90bd2fa438b9   (worker2)
Hostname: ad9f8613a454   (worker1)
Hostname: 90bd2fa438b9   (worker2)
...  6회 모두 성공
```

같은 시점에 manager의 eth0에서 VXLAN 트래픽을 떴습니다.

```
10.0.0.2.54702 > 10.0.0.10.8080: Flags [S]        ← manager ingress(10.0.0.2) → worker1 replica
10.0.0.10.8080 > 10.0.0.2.54702: Flags [S.]
10.0.0.2.54702 > 10.0.0.10.8080: HTTP: GET / HTTP/1.0
10.0.0.10.8080 > 10.0.0.2.54702: HTTP: HTTP/1.0 200 OK
```

manager가 처리할 수 없는 요청을 VXLAN으로 worker1에 넘기고 응답을 받아 오는 모습입니다. 이때 HAProxy에서는 manager도 **UP**이었고, 맥에서 30회 요청하면 replica가 있는 두 노드가 14회, 16회씩 응답했습니다.

(목적지 포트가 80이 아니라 8080으로 보이는 건, swarm이 8080 → 80 변환을 replica 쪽 네임스페이스 안에서 하기 때문입니다.)

### 두 단계 분산: Hostname 순서가 섞이는 이유

이 구성에서는 분산이 두 번 일어납니다.

```
                  curl localhost:8080
                          │
                          ▼
                 ┌──────────────────┐
  1단계          │ HAProxy          │  "어느 노드로?"  manager → worker1 → worker2 순서
                 └──┬──────┬──────┬─┘
                    ▼      ▼      ▼
                 manager worker1 worker2
  2단계           IPVS    IPVS    IPVS     "어느 replica로?"  A → B → C 순서
                                          (노드마다 순번을 따로 셈)
```

각 노드의 IPVS는 순번을 **따로** 셉니다. 그래서 다음과 같이 같은 replica가 연달아 나올 수 있습니다.

```
요청  HAProxy 선택   그 노드 IPVS의 순번    응답
 #1   → manager      manager: [A] B  C     → A
 #2   → worker1      worker1: [A] B  C     → A   ← 또 A
 #3   → worker2      worker2: [A] B  C     → A   ← 또 A
 #4   → manager      manager:  A [B] C     → B
```

또 HAProxy의 health check(2초마다)도 IPVS를 거치면서 순번을 하나씩 가져가므로 순서가 더 섞입니다. 하지만 두 단계 모두 돌아가며 고르기 때문에, 요청이 쌓이면 몫은 비슷해집니다(30회 요청 시 9/10/11회).

### 참고: host 모드 publish (routing mesh 우회)

노드 간 이동 자체를 없애는 방법도 있습니다. **이 실습에서는 적용하지 않았고, 개념만 정리합니다.**

```bash
docker service create --name web --mode global \
  --publish mode=host,target=80,published=8080 traefik/whoami
```

host 모드에서는 노드의 8080이 IPVS가 아니라 **그 노드의 컨테이너 하나에 직결**됩니다(`docker run -p`와 같은 방식). 다른 노드의 replica를 모르므로 넘길 수도 없습니다.

| | ingress 모드 (이 실습) | host 모드 |
|---|---|---|
| 요청 경로 | 노드 → (상당수) 다른 노드의 replica | 노드 → 자기 노드의 replica |
| 분산 | 외부 LB + IPVS 두 단계 | 외부 LB 한 단계 |
| replica 없는 노드 | 응답함 | 응답 없음 → 외부 LB health check 필수 |
| 한 노드에 replica 여러 개 | 가능 | 불가 (포트 충돌) → 보통 `--mode global` |
| 노드 장애 시 replica 재생성 | 다른 노드에 다시 띄움 | 안 함 (노드 수만큼만 운영) |

---

## 트러블슈팅 1: curl이 3번 중 1번만 성공 (VXLAN 체크섬)

### 증상

- 처음에는 manager에 `ports: "8080:8080"`을 직접 매핑했습니다.
- 맥에서 `curl localhost:8080`을 반복하면 **3번 중 1번만 응답하고, 나머지는 응답 없이 멈췄습니다**(timeout).
- 성공한 응답은 **항상 manager에 있는 replica**였습니다.
- 반면 **manager 안에서** `docker exec manager wget -qO- localhost:8080`으로 요청하면 세 replica 모두 정상 응답했습니다. overlay 네트워크 자체는 정상이었습니다.

[routing mesh 동작](#routing-mesh의-동작)에 대입하면, 로컬 replica로 가는 요청은 되고 **VXLAN으로 다른 노드에 넘어가는 요청만 실패**한다는 뜻입니다.

### 진단: tcpdump로 패킷 추적

노드 컨테이너 안에 `apk add tcpdump`로 설치해서 확인했습니다.

**1) manager**: 맥에서 온 SYN이 VXLAN에 담겨 worker2로 나가는 것까지는 보였지만, SYN-ACK가 돌아오지 않았습니다.

```
172.30.0.10.60544 > 172.30.0.12.4789: VXLAN, vni 4096
10.0.0.2.60931 > 10.0.0.7.8080: Flags [S] ...        ← 재전송만 반복
```

**2) worker2**: VXLAN 패킷이 eth0에 도착은 했지만, 컨테이너 쪽으로 전달되지 않았습니다. `-vv`로 체크섬을 보니 틀려 있었습니다.

```
172.30.0.10.34976 > 172.30.0.12.4789: [bad udp cksum 0x58be -> 0xa53a!] VXLAN ...
10.0.0.2.30749 > 10.0.0.7.8080: Flags [S], cksum 0x99c9 (incorrect -> 0x47d8)
```

**3) 거슬러 올라가 manager eth0에 처음 들어온 패킷**: 맥에서 온 패킷은 **처음부터 TCP 체크섬이 `0x0000`**이었습니다.

```
192.168.65.1.46611 > 172.30.0.10.8080: Flags [S], cksum 0x0000 (incorrect -> 0x5241)
```

### 원인

**확인한 사실**
- Docker Desktop 포트포워딩을 거쳐 들어온 패킷은 TCP 체크섬이 `0x0000`이었습니다.
- 그 패킷이 같은 노드의 replica로 가면 정상 처리됐습니다.
- 다른 노드로 VXLAN 전달되면 수신 노드에서 체크섬 오류로 처리되지 않았습니다.

**동작 원리 (추정)**
- Docker Desktop은 맥의 연결을 받아 Linux VM 안으로 패킷을 다시 넣는데, 이때 체크섬을 계산하지 않고 "검증 완료" 표시(checksum offload 관련 플래그)만 붙이는 것으로 보입니다.
- 같은 커널 안에서 로컬로 전달되면 커널이 그 표시를 믿고 넘어가므로 문제가 없습니다.
- VXLAN으로 감싸 다른 노드로 보내면, 수신 노드 커널은 표시 없이 새로 받은 패킷이라 체크섬을 실제로 검사하고 버립니다.

**정리**
- DinD 자체의 문제라기보다 **Docker Desktop 포트포워딩과 swarm VXLAN이 만나는 지점**의 문제입니다.
- 리눅스에 Docker Engine을 직접 설치한 환경이라면 중간에 이런 포트포워딩 계층이 없으니 생기지 않을 가능성이 높습니다 **(추정, 직접 확인하지 않음)**.

### 시도했지만 효과가 없었던 방법

모두 노드 컨테이너 안에서만 적용했고, 맥이나 Docker Desktop 설정은 바꾸지 않았습니다.

| 시도 | 결과 |
|---|---|
| `ethtool -K <if> rx off tx off` (manager의 eth0, `ingress_sbox`, overlay 네임스페이스 인터페이스) | 효과 없음 |
| `ingress_sbox`에 `iptables -t mangle -A POSTROUTING -p tcp -j CHECKSUM --checksum-fill` | 규칙에 패킷이 매칭되지만 효과 없음 |

두 방법 모두 "체크섬을 나중에 계산할 예정"인 패킷에 작용합니다. 이번 패킷은 체크섬이 0인 채 이미 "검증 완료"로 표시되어 있어서 적용 대상이 아니었던 것으로 보입니다 **(추정)**.

### 해결: 앞단에 프록시를 두고 TCP 연결을 새로 맺는다

```
[변경 전]  Mac → Docker Desktop → manager:8080 → ingress ─VXLAN─▶ ✗ (체크섬 0)

[변경 후]  Mac → Docker Desktop → lb ─[새 TCP 연결]→ 노드:8080 → ingress ─VXLAN─▶ ✓
```

- manager의 `ports`를 제거하고, `lb` 컨테이너가 `8080:8080`을 받게 했습니다.
- lb는 맥의 연결을 자기 쪽에서 끝내고, 노드로 **새 연결**을 엽니다. 새 연결의 패킷은 lb 컨테이너의 커널 네트워크 스택이 정상적으로 만들기 때문에 VXLAN을 거쳐도 버려지지 않습니다.
- 결과적으로 `curl localhost:8080`이 매번 성공하고, 세 replica가 모두 응답합니다.

### lb 변경: socat → HAProxy

1. **socat:** 처음에는 가장 단순한 TCP 파이프로 해결했습니다(`tcp-listen:8080,fork tcp-connect:172.30.0.10:8080`). 체크섬 문제는 이것만으로 해결됐지만, **manager로만** 연결하는 구조라 manager가 죽으면 worker replica가 살아 있어도 외부 접속이 끊깁니다.
2. **HAProxy:** 세 노드로 분산하고 health check를 붙였습니다. [공식 문서의 권장 구성](https://docs.docker.com/engine/swarm/ingress/)과 같은 형태입니다. 효과는 [노드 장애 실습](#노드-장애-실습)에서 확인할 수 있습니다.

| | socat | HAProxy |
|---|---|---|
| 역할 | 1:1 TCP 파이프 | 로드밸런서 |
| 목적지 | manager 하나 | 세 노드 |
| 노드 장애 | manager 장애 시 접속 불가 | health check로 자동 제외 |
| 모니터링 | 없음 | 통계 페이지 |

---

## 트러블슈팅 2: 재시작 후 정상화까지 너무 오래 걸림

볼륨을 유지한 채 `docker compose down && docker compose up -d`로 재시작한 뒤, 세 replica가 모두 응답하기까지 걸린 시간입니다.

| 단계 | 초기 | 원인 A 수정 후 | 원인 B 수정 후 (최종) |
|---|---|---|---|
| manager dockerd 응답 | 미측정 | 17초 | 2초 |
| 노드 3개 Ready | 미측정 | 34초 | 4초 |
| 서비스 3/3 | 미측정 | 39초 | 9초 |
| **세 replica 모두 응답** | **약 1분 45초** | **40초** | **10초** (HAProxy 적용 후 11초) |

### 원인 A: gossip 접속 순서 경쟁 (1분 45초 → 40초)

**증상**
- 재시작 직후 한동안 요청이 **manager replica로만** 갔습니다.
- manager의 IPVS에는 로컬 replica 1개만 등록돼 있었습니다.

```bash
$ docker exec manager nsenter --net=/var/run/docker/netns/ingress_sbox ipvsadm -Ln
FWM  256 rr
  -> 10.0.0.8:0     Masq    1      0          6        ← 1개뿐
```

**worker 로그**

```
Node 2c8faa3fcc3a/172.30.0.11, joined gossip cluster
Error in joining gossip cluster: join will be retried in background
  failed to join 172.30.0.10:7946: dial tcp 172.30.0.10:7946: connect: connection refused
```

**원인**
- 세 컨테이너가 동시에 뜨면서, worker가 manager의 gossip 포트(7946)가 열리기 **약 0.1초 전**에 접속을 시도해 실패했습니다.
- 로그 설정값의 `rejoinClusterInterval`이 60초였습니다. 실제로 약 1분 45초 뒤에 `joined gossip cluster` 로그와 함께 IPVS에 세 replica가 모두 등록됐습니다.
- gossip이 연결되기 전까지 manager는 worker에 있는 replica를 알 수 없었습니다.

**해결**
- manager에 `healthcheck`를 추가했습니다. 7946이 열릴 때까지 healthy가 되지 않습니다.
- worker에 `depends_on: condition: service_healthy`를 걸었습니다.

### 원인 B: dockerd의 의도적인 15초 시작 지연 (40초 → 10초)

**증상**
- 원인 A를 고친 뒤 구간별로 재 보니, dockerd가 응답하기까지 컨테이너마다 17초가 걸렸습니다.
- 로그를 보면 컨테이너 시작(04:34:28) 후 dockerd 초기화(04:34:44)까지 약 16초가 비어 있었고, 그 직전에 이런 경고가 있었습니다.

```
Binding to IP address without --tlsverify is deprecated.
Startup is intentionally being slowed down to show this message
You can override this by explicitly specifying '--tls=false' or '--tlsverify=false'
```

**원인**
- 당시 compose에는 `DOCKER_TLS_CERTDIR: ""`만 있고 `command`가 없었습니다.
- 이 경우 dind 이미지의 entrypoint(`dockerd-entrypoint.sh`)가 `--host=tcp://0.0.0.0:2375`를 자동으로 붙입니다.
- TLS 없이 TCP로 API를 열면 dockerd가 경고를 보여주려고 **일부러 시작을 늦춥니다.**
- 원인 A를 고치느라 worker가 manager를 기다리게 되면서, 이 지연이 **manager 한 번 + worker 한 번**으로 두 번 쌓였습니다.

**해결**
- `command: ["dockerd", "--host=unix:///var/run/docker.sock"]`로 TCP API를 아예 열지 않았습니다.
- 경고 메시지가 안내하는 `--tls=false`로도 지연을 끌 수 있지만, 쓰지 않는 인증 없는 root API를 열어 둘 이유가 없어서 닫는 쪽을 택했습니다.
- `command`를 직접 지정하면 entrypoint가 기본 인자를 붙이지 않으므로, 이후 `DOCKER_TLS_CERTDIR: ""`도 compose에서 뺐습니다. 빼고 다시 측정해도 재시작 후 세 replica가 모두 응답하기까지 11초였습니다.
- 이후 manager는 원격 접속을 위해 TLS(2376)를 켰습니다([manager 원격 접속](#manager-원격-접속-tls)). 지연은 **TLS 없이** TCP를 열 때만 생기므로 TLS에서는 없습니다. 지연 로그 0건, 노드 준비 3초, 재시작 후 세 replica 응답까지 11초로 측정됐습니다.

### 남은 약 5초

- 재시작 전에 떠 있던 task는 `Failed`(`task: non-zero exit (2)`)로 처리됩니다.
- swarm은 새 task를 띄우기 전에 기본 **restart delay 5초**를 기다립니다.
- 더 줄이려면 서비스를 만들 때 옵션을 주면 됩니다. 이 옵션은 적용해서 측정해 보지는 않았습니다.

```bash
docker exec manager docker service create --name web --replicas 3 -p 8080:80 --restart-delay 1s traefik/whoami
```

---

## 노드 장애 실습

HAProxy가 세 노드로 분산하므로, 노드 하나가 죽어도 외부 접속이 유지됩니다.

```bash
# 터미널 1: 요청을 계속 보냄
while true; do curl -s -m 3 localhost:8080 | grep Hostname || echo FAIL; sleep 0.5; done

# 터미널 2: manager 중지 → 통계 페이지(localhost:8404)에서 manager가 DOWN으로 바뀜
docker stop manager

# 복구
docker start manager
```

```
worker1이 죽었을 때 (manager도 같은 원리)

                    HAProxy
      ┌────────────────┼────────────────┐
      ▼                ✗                ▼
 manager:8080     worker1:8080     worker2:8080
                  (health check
                   실패 → 제외)
```

**실측 결과 (`docker stop manager`)**
- 0.5초 간격으로 23초 동안 보낸 요청 40회가 **모두 성공**했습니다.
- HAProxy 통계에서 manager가 DOWN으로 바뀌었고, 나머지 두 replica가 응답했습니다.
- `docker start manager` 후 약 11초 만에 manager가 다시 UP이 되고 서비스도 3/3으로 돌아왔습니다.

**알아둘 점**
- 이 구성은 **manager가 1대**입니다. manager가 멈춘 동안에는 swarm 관리 기능(`service create/update/scale`, 죽은 task 재배치)이 동작하지 않습니다. 이미 떠 있는 worker의 replica는 계속 응답합니다.
- manager 장애에도 관리 기능을 유지하려면 manager를 3대 이상 홀수로 두어 raft 과반수를 확보해야 합니다.

---

## manager 원격 접속 (TLS)

`docker exec manager docker ...` 없이, **컨테이너 밖에서** manager의 dockerd를 조작할 수 있게 했습니다. 나중에 CI 파이프라인에서 레지스트리(예: Harbor)로 이미지를 push하고 swarm에 `docker stack deploy`로 배포할 때 필요한 기반입니다.

```
Mac ──127.0.0.1:2376 (TLS)──────────────▶ manager dockerd
CI 컨테이너 ──tcp://manager:2376 (TLS)──▶ manager dockerd   (swarm-net에 붙은 경우)
                                         worker는 API 포트 없음
```

### 인증서는 어디서 만들어지나

Mac에서 만들어 컨테이너로 넣는 게 **아닙니다.** manager 컨테이너가 시작될 때 dind 이미지의 entrypoint(`dockerd-entrypoint.sh`)가 **컨테이너 안의 openssl로** 자동 생성합니다.

```
manager 컨테이너 시작
  └─ dockerd-entrypoint.sh
       ├─ CA 개인키/인증서 생성 (없을 때만 키 생성)
       ├─ 서버 키/인증서 생성  → dockerd가 사용
       └─ 클라이언트 키/인증서 생성
  └─ dockerd 시작 (TLS 2376)
```

파일은 **컨테이너 → Mac** 방향으로 나옵니다.

```
manager: /certs              ── named volume (manager-certs) ──  Docker 내부 저장소 (Mac에서 안 보임)
         ├── ca/     CA 개인키 (컨테이너 밖으로 나오지 않음)
         ├── server/ dockerd용
         └── client/ ══ bind mount ══▶ Mac: swarm-http/certs/manager-client/
                                          ├── ca.pem    (CA 인증서: 서버를 검증할 때)
                                          ├── cert.pem  (클라이언트 인증서)
                                          └── key.pem   (클라이언트 개인키)
```

bind mount는 양방향이라, Mac 쪽에 `key.pem`이 이미 있으면 컨테이너가 그 키를 재사용하고 `cert.pem`만 다시 서명합니다. 그래서 `down -v`로 CA가 바뀌어도 이 디렉터리는 자동으로 새 CA에 맞게 갱신됩니다.

### 인증서가 필요한 경우

| 접속 방식 | 경로 | 인증서 |
|---|---|---|
| `docker exec manager docker ...` | manager 안에서 unix 소켓 | **필요 없음** (빠른 시작의 기본 방식) |
| Mac에서 원격 조작 | `tcp://127.0.0.1:2376` | 필요 |
| 다른 컨테이너 (CI 등) | `tcp://manager:2376` | 필요 |

### 사용 방법

docker CLI에 인증서를 알려주는 방법은 네 가지입니다. 모두 `swarm-http/` 디렉터리에서 실행한다고 가정합니다.

| 방법 | 설정 범위 | 추천 상황 |
|---|---|---|
| ① 명령 플래그 | 그 명령 한 번 | 가끔 확인할 때 |
| ② 환경변수 | 터미널 세션 동안 | 한동안 manager만 다룰 때 |
| ③ docker context | Mac에 영구 등록 | 자주 쓸 때 |
| ④ 컨테이너 환경변수 | 컨테이너 설정 | CI 컨테이너 |

**① 명령 플래그**

```bash
C=$PWD/certs/manager-client
docker -H tcp://127.0.0.1:2376 --tlsverify \
  --tlscacert $C/ca.pem --tlscert $C/cert.pem --tlskey $C/key.pem \
  node ls
```

**② 환경변수 (터미널 세션 동안)**

```bash
export DOCKER_HOST=tcp://127.0.0.1:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$PWD/certs/manager-client   # ca.pem, cert.pem, key.pem이 있는 폴더

docker node ls        # 플래그 없이 manager로 감
docker service ls

unset DOCKER_HOST DOCKER_TLS_VERIFY DOCKER_CERT_PATH   # 끝나면 되돌리기
```

> 설정해 둔 동안에는 그 터미널의 **모든** docker 명령이 manager로 갑니다. `docker compose up`도 Docker Desktop이 아니라 manager 안의 dockerd로 가게 되니, 작업이 끝나면 꼭 `unset` 하세요.

**③ docker context (한 번 등록)**

```bash
C=$PWD/certs/manager-client
docker context create swarm-http --docker \
  "host=tcp://127.0.0.1:2376,ca=$C/ca.pem,cert=$C/cert.pem,key=$C/key.pem"

docker --context swarm-http node ls   # 이 명령만 manager로
docker context rm swarm-http          # 필요 없어지면 삭제
```

- 명령마다 `--context swarm-http`만 붙이면 되고, 나머지 명령은 평소처럼 Docker Desktop으로 갑니다.
- Mac의 `~/.docker/contexts`에 설정이 저장됩니다.
- `docker context use swarm-http`로 기본값을 바꿀 수도 있지만, ②와 같은 이유로 헷갈리기 쉬워 추천하지 않습니다.
- context는 인증서 파일 **내용**을 복사해 저장하는 것으로 알려져 있습니다 **(미검증)**. 그렇다면 `down -v`로 CA가 바뀐 뒤에는 context를 지우고 다시 만들어야 합니다.

**④ 컨테이너 환경변수 (CI 형태)**

swarm 네트워크에 붙이고 클라이언트 인증서를 읽기 전용으로 마운트합니다.

```bash
docker run --rm --network swarm-http_swarm-net \
  -v $PWD/certs/manager-client:/certs/client:ro \
  -e DOCKER_HOST=tcp://manager:2376 -e DOCKER_TLS_VERIFY=1 -e DOCKER_CERT_PATH=/certs/client \
  docker:29-cli docker service ls
```

compose나 CI 도구에서는 같은 환경변수를 설정 파일에 **한 번만** 넣으면, 컨테이너 안에서는 `docker stack deploy ...`처럼 평소대로 쓸 수 있습니다. [ci-dind](../ci-dind/README.md)의 `ci` 서비스가 이 방식입니다.

**검증 범위**
- ①과 ④: 이 실습에서 실행해 확인했습니다.
- ②: 같은 환경변수를 ④의 컨테이너 안에서 검증했고, Mac 터미널에서 직접 실행하지는 않았습니다.
- ③: Mac 설정이 바뀌므로 실행하지 않았습니다.

### 주의 사항

- `certs/`에는 **클라이언트 개인키**(`key.pem`)가 있습니다. 이 파일이 있으면 manager를 root 권한으로 조작할 수 있습니다. `.gitignore`에 포함되어 있으니 커밋하지 말고, 다른 곳에 복사할 때도 주의하세요.
- 인증서를 다른 곳(예: CI 서버)에 복사해 두었다면, `down -v` 후에는 CA가 바뀌므로 다시 복사해야 합니다([인증서 유지 방식](#인증서-유지-방식) 참고).

### 서버 인증서의 이름 (SAN)

entrypoint가 컨테이너 IP, hostname, `docker`, `localhost`를 자동으로 넣습니다. `hostname: manager`와 고정 IP 덕분에 별도 설정 없이 다음 이름으로 접속할 수 있습니다.

```
DNS:docker, DNS:localhost, DNS:manager, IP:127.0.0.1, IP:172.30.0.10, IP:::1
```

다른 이름(예: 도메인)으로 접속해야 하면 `DOCKER_TLS_SAN` 환경변수로 추가할 수 있습니다(entrypoint 코드 기준, 미검증). [ci-dind](../ci-dind/README.md)에서는 서비스 이름이 SAN에 없어 검증에 실패한 적이 있습니다.

### 인증서 유지 방식

entrypoint 코드를 확인한 동작입니다.
- **개인키**(CA, 서버, 클라이언트)는 파일이 있으면 재사용합니다.
- **인증서**(공개 부분)는 시작할 때마다 같은 키로 다시 서명합니다. IP나 이름 변경, 만료에 대비한 동작입니다.

따라서 `/certs`를 볼륨(`manager-certs`)에 두면, 컨테이너를 새로 만들어도 **신뢰 관계가 유지**됩니다.

| 시나리오 | 결과 (실측) |
|---|---|
| `down` → `up --force-recreate` | CA, 서버, 클라이언트 키 지문 동일. 인증서 serial만 바뀜 |
| 재생성 **전에** 복사해 둔 클라이언트 인증서로 접속 | **성공** |
| `down -v` 후 재시작 | CA가 새로 생성됨. `./certs/manager-client`는 자동 갱신되어 접속 성공 |
| `down -v` **전에** 복사해 둔 클라이언트 인증서로 접속 | **실패**: `x509: certificate signed by unknown authority` |

클라이언트 인증서를 다른 곳(예: CI 서버)에 복사해 두었다면, `down -v` 후에는 다시 복사해야 합니다.

### 검증 결과

| 확인 항목 | 결과 |
|---|---|
| manager dockerd 인자 | `--host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify ...` |
| worker dockerd 인자 | `--host=unix:///var/run/docker.sock` (변경 없음) |
| 15초 시작 지연 | 없음 (로그 0건, 노드 준비 3초) |
| Mac에서 `docker -H tcp://127.0.0.1:2376 --tlsverify ... node ls` | 성공 (노드 3개 Ready) |
| swarm-net 컨테이너에서 `tcp://manager:2376`로 `service ls` | 성공 |
| 클라이언트 인증서 없이 접속 (Mac, swarm-net 모두) | **거부** (TLS 핸드셰이크 실패, curl exit 56) |
| worker의 2375, 2376 포트 | 닫혀 있음 |
| 포트 공개 범위 (`docker port manager`) | `2376/tcp -> 127.0.0.1:2376` |
| 서비스 동작 (30회 요청) | 세 replica에 10/10/10회 |

### 앞으로 확장할 때

- **CI 컨테이너에서 배포:** swarm-net에 붙이고 `./certs/manager-client`를 마운트하면 `docker stack deploy`를 할 수 있습니다(위 사용법의 마지막 예시).
- **레지스트리(Harbor 등):** swarm 노드가 레지스트리에서 이미지를 pull하려면, 노드의 dockerd가 그 레지스트리를 신뢰해야 합니다. 레지스트리의 TLS 인증서 또는 insecure-registry 설정이 필요한데, 이는 이번 Docker API 인증서와는 **별개의 인증서**입니다.
- **manager를 여러 대로 늘릴 때:** 각 manager에 같은 방식으로 TLS를 켜고, 인증서를 어떻게 공유할지(CA를 하나로 통일할지)를 정해야 합니다.

---

## 디버깅 명령 모음

```bash
# ingress 로드밸런서(IPVS)에 등록된 replica 확인
docker exec manager apk add -q ipvsadm
docker exec manager nsenter --net=/var/run/docker/netns/ingress_sbox ipvsadm -Ln

# VXLAN 트래픽과 체크섬 확인
docker exec worker1 apk add -q tcpdump
docker exec worker1 tcpdump -vvni eth0 udp port 4789

# gossip join 실패 여부
docker logs worker1 2>&1 | grep -i gossip

# manager 헬스 상태
docker inspect -f '{{.State.Health.Status}}' manager

# HAProxy가 보는 노드 상태
curl -s 'localhost:8404/;csv' | awk -F, '$1=="swarm_nodes"{print $2, $18}'

# 특정 노드의 8080으로 직접 요청 (HAProxy 우회)
docker exec lb wget -qO- http://172.30.0.10:8080

# manager 서버 인증서의 SAN과 키 지문
docker exec manager openssl x509 -in /certs/server/cert.pem -noout -ext subjectAltName
docker exec manager sh -c 'openssl pkey -in /certs/ca/key.pem -pubout | sha256sum'
```

`apk add`로 설치한 도구는 노드 컨테이너가 다시 만들어지면 사라집니다.

---

## 참고 자료

- [Use swarm mode routing mesh (Docker Docs)](https://docs.docker.com/engine/swarm/ingress/): routing mesh 동작, HAProxy 외부 LB 예시, host 모드(Bypass the routing mesh)
