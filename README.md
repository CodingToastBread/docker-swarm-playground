# swarm-lab

맥북(Docker Desktop) 한 대에서 **Docker-in-Docker(DinD)** 컨테이너 3개로 Docker Swarm 클러스터(manager 1 + worker 2)를 구성해 실습하는 환경입니다.

```
 Mac (localhost:8080 서비스, localhost:8404 HAProxy 통계)
        │  Docker Desktop 포트포워딩
        ▼
 ┌──────────────┐        swarm-net (172.30.0.0/24, compose 브리지 네트워크)
 │ lb (HAProxy) │ 172.30.0.2
 └──────┬───────┘
        │ 새 TCP 연결, 세 노드로 라운드로빈 + health check
        ├──────────────────┬──────────────────┐
        ▼                  ▼                  ▼
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ manager      │   │ worker1      │   │ worker2      │
 │ 172.30.0.10  │   │ 172.30.0.11  │   │ 172.30.0.12  │
 │ dockerd      │   │ dockerd      │   │ dockerd      │
 │  └ web.x     │   │  └ web.x     │   │  └ web.x     │
 └──────────────┘   └──────────────┘   └──────────────┘
        ▲                  ▲                  ▲
        └──── swarm overlay (VXLAN, UDP 4789) ┘
```

각 노드 컨테이너 안에는 독립된 `dockerd`가 떠 있습니다. 이 dockerd들이 서로 swarm을 이루고, 서비스 컨테이너(`web.x`)는 **노드 컨테이너 안에서** 실행됩니다.

---

## 빠른 시작

```bash
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

> Hostname이 `A → B → C` 순서로 딱딱 돌지 않고 섞여 나올 수 있습니다. HAProxy가 노드를 고르고, 그 노드의 ingress(IPVS)가 replica를 **한 번 더** 고르는 두 단계 분산이라서 그렇습니다. 여러 번 요청하면 고르게 분산됩니다(30회 요청 시 9/10/11회).

> **Driver가 `overlay2`가 아니라 `overlayfs`로 나오는 이유**
> Docker 29부터 이미지 저장을 containerd snapshotter가 맡기 때문에 `overlayfs`로 표시됩니다. 정상입니다.

### 스택 배포 (선택)

`./stacks`는 manager의 `/stacks`에 마운트되어 있습니다.

```bash
docker exec manager docker stack deploy -c /stacks/whoami.yml demo
```

`stacks/whoami.yml`도 8080 포트를 사용하므로, 위의 `web` 서비스와 동시에 띄우려면 한쪽을 먼저 지우세요(`docker exec manager docker service rm web`).

### 재시작 / 초기화

```bash
docker compose down && docker compose up -d   # 볼륨 유지: swarm과 서비스가 그대로 살아남 (약 10초 후 정상 응답)
docker compose down -v                        # 볼륨까지 삭제: swarm을 처음부터 다시 구성
```

---

## docker-compose.yml 구성 요소 설명

### 공통: 노드 서비스 (`manager`, `worker1`, `worker2`)

| 항목 | 값 | 설명 / 필요한 이유 |
|---|---|---|
| `image` | `docker:29-dind` | 컨테이너 안에서 dockerd를 실행하는 공식 DinD 이미지입니다. 컨테이너 하나가 "Docker가 설치된 서버 한 대" 역할을 합니다. |
| `container_name` | `manager` 등 | `docker exec manager ...`처럼 고정된 이름으로 접근하려고 지정했습니다. |
| `hostname` | `manager` 등 | swarm은 노드 이름으로 hostname을 씁니다. 지정하지 않으면 `docker node ls`에 무작위 컨테이너 ID가 나옵니다. |
| `privileged` | `true` | 안쪽 dockerd가 네트워크 네임스페이스, iptables, VXLAN 인터페이스, cgroup, overlay 마운트를 만들려면 호스트 커널 권한이 필요합니다. DinD의 필수 조건입니다. |
| `environment.DOCKER_TLS_CERTDIR` | `""` | 비워 두면 dind 이미지가 TLS 인증서를 만들지 않습니다. 실습 환경이라 TLS가 필요 없습니다. |
| `command` | `["dockerd", "--host=unix:///var/run/docker.sock"]` | dockerd가 **unix 소켓만** 열도록 합니다. 이 설정이 없으면 이미지가 `tcp://0.0.0.0:2375`(인증 없는 TCP API)를 자동으로 붙이는데, 이때 dockerd가 경고를 보여주려고 **시작을 약 15초 지연**시킵니다. 모든 제어를 `docker exec`(unix 소켓)로 하므로 TCP API는 필요 없고, 보안상으로도 닫는 편이 낫습니다. 자세한 내용은 [트러블슈팅 2](#2-재시작-후-정상화까지-너무-오래-걸림)를 참고하세요. |
| `networks.swarm-net.ipv4_address` | `172.30.0.10/11/12` | IP를 고정합니다. `swarm init --advertise-addr`와 `swarm join`에 쓰는 주소이고, 재시작 후에도 같은 IP여야 기존 swarm 상태(볼륨에 저장됨)가 그대로 유효합니다. |
| `volumes` | `<node>-data:/var/lib/docker` | 안쪽 dockerd의 데이터(이미지, 컨테이너, **swarm raft 상태**)를 named volume에 보관합니다. 이렇게 하면 `compose down/up` 후에도 swarm과 서비스가 유지되고 이미지를 다시 받지 않습니다. 또한 `/var/lib/docker`가 컨테이너 자체의 overlay 파일시스템 위에 있으면 overlay 위에 overlay를 겹치게 되어 동작하지 않으므로, 별도 볼륨이 필요합니다. |

### manager 전용

| 항목 | 설명 / 필요한 이유 |
|---|---|
| `volumes: ./stacks:/stacks` | 호스트의 스택 파일을 manager 안에서 `docker stack deploy -c /stacks/...`로 쓰기 위한 마운트입니다. |
| `healthcheck` | dockerd가 응답하고, swarm에 참여한 상태라면 **gossip 포트 7946까지 열린 뒤에** healthy로 판정합니다. swarm을 아직 만들지 않은 처음 상태에서는 dockerd만 떠 있으면 통과합니다. worker의 `depends_on`과 함께 쓰입니다([트러블슈팅 2](#2-재시작-후-정상화까지-너무-오래-걸림)). |

healthcheck 명령 풀이:

```sh
docker info >/dev/null 2>&1 &&                                   # dockerd가 응답하는지
{ [ "$(docker info --format '{{.Swarm.LocalNodeState}}')" != active ]   # swarm 전이면 통과
  || nc -z 127.0.0.1 7946; }                                      # swarm 상태면 7946이 열려야 통과
```

compose 파일 안에서는 `$`를 `$$`로 이스케이프해야 compose 변수 치환이 일어나지 않습니다.

### worker 전용

| 항목 | 설명 / 필요한 이유 |
|---|---|
| `depends_on.manager.condition: service_healthy` | manager가 healthy가 된 뒤에 worker를 시작합니다. 재시작할 때 worker가 manager의 gossip보다 먼저 떠서 join에 실패하는 경쟁 상태를 막습니다. |

### `lb` 서비스

| 항목 | 값 | 설명 / 필요한 이유 |
|---|---|---|
| `image` | `haproxy:3.2-alpine` | 외부 로드밸런서입니다. 맥에서 온 연결을 **여기서 끊고**, 노드 쪽으로 **새 TCP 연결**을 맺습니다. |
| `ports` | `8080:8080` | 맥의 `localhost:8080`을 lb로 연결합니다. **노드 컨테이너에 직접 포트를 매핑하지 않는 것**이 핵심입니다([트러블슈팅 1](#1-curl-localhost8080이-3번-중-1번만-성공-vxlan-체크섬-문제)). |
| `ports` | `8404:8404` | HAProxy 통계 페이지입니다. 노드별 UP/DOWN, 요청 수를 볼 수 있습니다. |
| `volumes` | `./haproxy/haproxy.cfg:...:ro` | HAProxy 설정 파일을 읽기 전용으로 마운트합니다. |
| `ipv4_address` | `172.30.0.2` | 노드 IP 대역과 겹치지 않게 고정했습니다. |
| `depends_on` | `manager`, `worker1`, `worker2` | 노드들이 뜬 다음에 시작합니다. 먼저 떠도 health check가 노드를 DOWN으로 두었다가 살아나면 UP으로 바꾸므로 필수는 아닙니다. |

### `haproxy/haproxy.cfg`

| 설정 | 설명 / 필요한 이유 |
|---|---|
| `frontend web` / `bind :8080` | 서비스 요청을 받는 입구입니다. |
| `backend swarm_nodes` / `server ... 172.30.0.1x:8080` | 세 노드의 published port(8080)입니다. routing mesh 덕분에 **어느 노드로 보내도** 서비스가 응답합니다. |
| `balance roundrobin` | 세 노드에 번갈아 보냅니다. |
| `option httpchk GET /` + `check inter 2s fall 2 rise 2` | 2초마다 각 노드로 HTTP 요청을 보내 상태를 확인합니다. 2번 연속 실패하면 DOWN(제외), 2번 연속 성공하면 UP(복귀)입니다. 포트가 열렸는지만 보는 TCP 체크와 달리 **실제 서비스 응답까지** 확인합니다. |
| `retries 2` + `option redispatch` | 연결에 실패하면 **다른 노드로** 재시도합니다. health check가 장애를 감지하기 전(최대 약 4초)의 요청도 실패하지 않게 해 줍니다. |
| `timeout connect 2s` | 죽은 노드로의 연결 시도를 빨리 포기하고 재시도로 넘어가게 합니다. |
| `http-reuse never` | 백엔드 연결을 요청 간에 재사용하지 않습니다. 재사용하면 한 연결이 노드 IPVS의 같은 replica에 고정되어 분산이 한쪽으로 쏠립니다. |
| `frontend stats` / `bind :8404` | 통계 페이지입니다(2초마다 자동 새로고침). |

### `networks` / `volumes`

| 항목 | 설명 |
|---|---|
| `networks.swarm-net` (subnet `172.30.0.0/24`) | 노드끼리 통신하는 "물리 네트워크" 역할을 하는 compose 브리지 네트워크입니다. 고정 IP를 쓰려면 subnet을 명시해야 합니다. swarm의 2377(관리), 7946(gossip), 4789/udp(VXLAN) 트래픽이 모두 이 네트워크를 지나갑니다. |
| `volumes` | 노드별 `/var/lib/docker`용 named volume입니다. `docker compose down -v`로 삭제됩니다. |

### 노드 내부에서 사용하는 포트

| 포트 | 용도 |
|---|---|
| 2377/tcp | swarm 관리(raft, join) |
| 7946/tcp+udp | gossip: 노드끼리 "어느 노드에 어떤 task/엔드포인트가 있는지"를 주고받습니다 |
| 4789/udp | VXLAN: overlay 네트워크의 실제 데이터 트래픽 |

compose 네트워크 안에서는 서로 모두 열려 있으므로 별도 설정이 필요 없습니다.

---

## 트러블슈팅

### 1. `curl localhost:8080`이 3번 중 1번만 성공 (VXLAN 체크섬 문제)

#### 증상

- 초기 구성은 manager에 `ports: "8080:8080"`을 직접 매핑하는 방식이었습니다.
- `curl localhost:8080`을 반복하면 한 번은 응답하고, 나머지 두 번은 응답 없이 멈췄습니다(timeout).
- 성공하는 응답의 Hostname은 **항상 manager에 떠 있는 replica**였습니다.
- 반면 manager 안에서 `docker exec manager wget -qO- localhost:8080`으로 요청하면 **세 replica 모두 정상 응답**했습니다. overlay 네트워크 자체는 정상이었다는 뜻입니다.

#### 배경: swarm ingress(routing mesh)의 동작

1. 요청이 어느 노드의 8080으로 들어오든, 그 노드의 `ingress_sbox`에 있는 IPVS 로드밸런서가 replica 하나를 고릅니다(라운드로빈).
2. 고른 replica가 **같은 노드**에 있으면 로컬에서 바로 전달합니다.
3. **다른 노드**에 있으면 패킷을 VXLAN(UDP 4789)으로 감싸 그 노드로 보냅니다.

3개 중 1개만 성공했다는 건 "로컬 replica는 되고, VXLAN을 거치는 경우만 실패한다"는 뜻입니다.

#### 진단 과정

manager와 worker2에서 tcpdump로 패킷을 추적했습니다. tcpdump는 노드 컨테이너 안에 `apk add tcpdump`로 설치했습니다.

1. **manager**: 맥에서 온 SYN이 VXLAN에 담겨 worker2(172.30.0.12:4789)로 **나가는 것**까지는 확인됐습니다. 하지만 SYN-ACK가 돌아오지 않았습니다.
2. **worker2**: 그 VXLAN 패킷이 eth0에 **도착은 했지만**, overlay 안쪽(컨테이너)으로 전달되지 않았습니다.
3. `tcpdump -vv`로 체크섬을 확인했습니다.
   ```
   # worker2에서 수신한 VXLAN 패킷
   172.30.0.10.34976 > 172.30.0.12.4789: [bad udp cksum ...] VXLAN ...
   10.0.0.2.30749 > 10.0.0.7.8080: Flags [S], cksum 0x99c9 (incorrect -> 0x47d8)
   ```
4. 거슬러 올라가 manager eth0에 **처음 들어온 패킷**을 확인했습니다.
   ```
   192.168.65.1.46611 > 172.30.0.10.8080: Flags [S], cksum 0x0000 (incorrect -> 0x5241)
   ```
   맥에서 온 패킷은 **처음부터 TCP 체크섬이 `0x0000`(계산되지 않음)** 상태였습니다.

#### 원인

- Docker Desktop의 포트포워딩은 맥의 연결을 받아 Linux VM 안으로 패킷을 다시 주입합니다. 이때 TCP 체크섬을 채우지 않고 "체크섬은 이미 검증됨"이라는 표시(checksum offload 플래그)만 붙여 넘깁니다.
- **로컬 전달**의 경우: 같은 커널 안에서 처리되고 커널은 이 표시를 믿으므로, 체크섬이 0이어도 문제가 없습니다. 3번 중 1번 성공한 경우가 이것입니다.
- **VXLAN 전달**의 경우: IPVS가 주소만 바꾸고 체크섬을 부분 갱신한 뒤, 패킷을 그대로 VXLAN에 감싸 다른 노드로 보냅니다. 수신 노드의 커널은 이 패킷을 새로 받은 것이라 체크섬을 실제로 검사하고, 틀렸으므로 **조용히 버립니다(drop)**. 클라이언트는 SYN-ACK를 받지 못해 응답 없이 기다리게 됩니다.

즉 **DinD의 문제가 아니라, Docker Desktop 포트포워딩과 swarm VXLAN의 조합에서 생기는 문제**입니다. 실제 서버에서는 NIC나 커널 스택을 거친 패킷이 들어오므로 이런 일이 없습니다.

#### 시도했지만 효과가 없었던 방법

모두 노드 컨테이너 안에서만 적용했고, 맥이나 Docker Desktop 설정은 바꾸지 않았습니다.

| 시도 | 결과 | 이유 |
|---|---|---|
| `ethtool -K eth0 rx off tx off` (manager, `ingress_sbox`, overlay 네임스페이스의 인터페이스) | 효과 없음 | 체크섬이 이미 0인 채로 "검증됨" 표시가 붙어서 들어오므로, 이후 단계의 offload를 꺼도 다시 계산되지 않습니다. |
| `ingress_sbox`에 `iptables -t mangle ... -j CHECKSUM --checksum-fill` | 규칙에 매칭은 되지만 효과 없음 | `CHECKSUM` 타깃은 "나중에 계산 예정(partial)" 상태의 패킷만 채웁니다. "검증됨" 상태의 패킷은 건드리지 않습니다. |

#### 해결: 외부 프록시(lb)를 앞에 둔다

```
Mac → Docker Desktop → lb ─[새 TCP 연결]→ 노드:8080 → ingress → 모든 replica
```

- manager의 `ports`를 제거하고, `lb` 컨테이너가 `8080:8080`을 받도록 바꿨습니다.
- lb는 맥에서 온 TCP 연결을 **자기 쪽에서 끝내고**, 노드로 **새 연결**을 엽니다. 새 연결의 패킷은 lb 컨테이너의 커널이 정상적으로 만든 것이라 체크섬이 올바르므로, VXLAN을 거쳐도 버려지지 않습니다.
- 실제 운영에서 swarm 앞에 외부 로드밸런서를 두는 구조와 같은 모양입니다.

결과: `curl localhost:8080`이 매번 성공하고, 세 replica가 모두 응답합니다.

#### lb 변경 이력: socat → HAProxy

처음에는 가장 단순한 TCP 파이프인 **socat**(`tcp-listen:8080,fork tcp-connect:172.30.0.10:8080`)으로 해결했습니다. 체크섬 문제는 해결됐지만 **manager로만** 연결하는 구조라, manager 컨테이너가 죽으면 worker replica가 살아 있어도 외부 접속이 끊기는 구조였습니다.

그래서 **HAProxy**로 바꿔 세 노드로 분산하고 health check를 붙였습니다. 아래 "노드 장애 실습"에서 효과를 확인할 수 있습니다.

---

### 2. 재시작 후 정상화까지 너무 오래 걸림

`docker compose down && docker compose up -d`(볼륨 유지)로 재시작한 뒤, 세 replica가 모두 응답하기까지 걸린 시간은 다음과 같습니다.

| 단계 | 초기 | gossip 수정 후 | 최종 |
|---|---|---|---|
| manager dockerd 응답 | - | 17초 | 2초 |
| 노드 3개 Ready | - | 34초 | 4초 |
| 서비스 3/3 | - | 39초 | 9초 |
| **세 replica 모두 응답** | **약 1분 45초** | **40초** | **10초** |

#### 원인 A: gossip 접속 순서 경쟁 (1분 45초 → 40초)

- 증상: 재시작 직후 한동안 요청이 **manager replica로만** 갔습니다. manager의 IPVS 백엔드에 로컬 task 1개만 등록돼 있었습니다(`ipvsadm -Ln`으로 확인).
- worker 로그:
  ```
  Error in joining gossip cluster: join will be retried in background
  failed to join 172.30.0.10:7946: dial tcp 172.30.0.10:7946: connect: connection refused
  ```
- 세 컨테이너가 동시에 뜨면서 worker가 manager의 gossip 포트(7946)가 열리기 **약 0.1초 전**에 접속을 시도해 실패했습니다.
- gossip 재접속은 **60초 주기**라서, 그동안 manager는 worker에 있는 replica를 알지 못했습니다.
- 해결: manager에 `healthcheck`(7946이 열릴 때까지 대기)를 추가하고, worker에 `depends_on: condition: service_healthy`를 걸었습니다.

#### 원인 B: dockerd의 의도적인 15초 시작 지연 (40초 → 10초)

- 컨테이너가 시작된 뒤 dockerd가 실제로 뜨기까지 16초 동안 로그가 없었습니다. 그 직전 로그는 다음과 같습니다.
  ```
  Binding to IP address without --tlsverify is deprecated.
  Startup is intentionally being slowed down to show this message
  ```
- `DOCKER_TLS_CERTDIR: ""`이면 dind entrypoint가 `--host=tcp://0.0.0.0:2375`(TLS 없는 TCP API)를 자동으로 추가합니다. 이 경우 dockerd가 경고를 보여주려고 시작을 약 15초 늦춥니다.
- 원인 A의 수정으로 worker가 manager를 기다리게 되면서, 이 지연이 **manager 15초 + worker 15초**로 두 번 쌓였습니다.
- 해결: `command: ["dockerd", "--host=unix:///var/run/docker.sock"]`로 TCP API를 열지 않게 했습니다. 모든 조작은 `docker exec`(unix 소켓)로 하므로 영향이 없습니다.

#### 남은 약 5초

재시작 전에 떠 있던 task는 `Failed`로 처리되고, swarm이 새 task를 띄우기 전에 기본 **restart delay 5초**를 기다립니다. 더 줄이고 싶으면 서비스를 만들 때 옵션을 추가하세요.

```bash
docker exec manager docker service create --name web --replicas 3 -p 8080:80 --restart-delay 1s traefik/whoami
```

---

## 노드 장애 실습

HAProxy가 세 노드로 분산하므로, 노드 하나가 죽어도 외부 접속이 유지됩니다.

```bash
# 요청을 계속 보내는 터미널
while true; do curl -s -m 3 localhost:8080 | grep Hostname || echo FAIL; sleep 0.5; done

# 다른 터미널에서 manager 중지 → 통계 페이지(localhost:8404)에서 manager가 DOWN으로 바뀜
docker stop manager

# 복구
docker start manager
```

실측 결과:
- manager를 멈춘 동안 0.5초 간격으로 보낸 요청 40회(23초)가 **전부 성공**했습니다. manager replica를 뺀 나머지 두 replica가 응답했습니다.
- `docker start manager` 후 약 11초 만에 HAProxy에서 manager가 UP으로 돌아오고 서비스도 3/3이 됐습니다.

알아둘 점:
- 이 구성은 **manager가 1대뿐**입니다. manager가 멈춘 동안에는 swarm 관리(`service create/update/scale`, 죽은 task 재배치)가 불가능합니다. 이미 떠 있는 worker의 task는 계속 서비스합니다.
- manager 장애에도 관리 기능을 유지하려면 manager를 3대(raft 과반수)로 구성해야 합니다.

---

## 유용한 디버깅 명령

```bash
# ingress 로드밸런서(IPVS)에 등록된 백엔드 확인
docker exec manager apk add -q ipvsadm
docker exec manager nsenter --net=/var/run/docker/netns/ingress_sbox ipvsadm -Ln

# VXLAN 트래픽과 체크섬 확인
docker exec worker1 apk add -q tcpdump
docker exec worker1 tcpdump -vvni eth0 udp port 4789

# gossip join 실패 여부
docker logs worker1 2>&1 | grep -i gossip

# manager 헬스 상태
docker inspect -f '{{.State.Health.Status}}' manager

# HAProxy가 보는 노드 상태 (CSV)
curl -s 'localhost:8404/;csv' | awk -F, '$1=="swarm_nodes"{print $2, $18}'
```
