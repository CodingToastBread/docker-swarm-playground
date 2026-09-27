# swarm-http

맥북(Docker Desktop) 한 대에서 DinD 컨테이너 3개로 Docker Swarm(manager 1 + worker 2)을 만들고, HAProxy를 앞에 둬서 HTTP 서비스를 분산하는 실습입니다.

- 테스트 환경: macOS 27.0 (arm64), Docker Desktop 29.8.0, `docker:29-dind` (29.8.1), `haproxy:3.2-alpine`
- 서비스 이미지: `traefik/whoami` (응답에 처리한 컨테이너의 Hostname이 나옴)

## 구조

```
 Mac ── localhost:8080 (서비스)  localhost:8404 (HAProxy 통계)
  │
  ▼
 lb (HAProxy, 172.30.0.2) ── 세 노드로 라운드로빈 + health check
  ├──────────────────┬──────────────────┐
  ▼                  ▼                  ▼
 manager            worker1            worker2         ← docker:29-dind 컨테이너
 172.30.0.10        172.30.0.11        172.30.0.12     ← 안쪽 dockerd끼리 swarm 구성
  └ web.x            └ web.x            └ web.x        ← 서비스는 노드 "안"에서 실행
  └────────── swarm overlay (VXLAN) ────┘
```

```
swarm-http/
├── docker-compose.yml     # 노드 3개 + lb
├── haproxy/haproxy.cfg
└── stacks/whoami.yml      # stack 예제 (manager의 /stacks에 마운트)
```

## 빠른 시작

```bash
cd swarm-http
docker compose up -d                                      # manager, worker1, worker2, lb 실행
docker exec manager docker info                           # 에러 없으면 dockerd 준비 완료

# 1. swarm 구성
docker exec manager docker swarm init --advertise-addr 172.30.0.10
TOKEN=$(docker exec manager docker swarm join-token -q worker)
docker exec worker1 docker swarm join --token $TOKEN 172.30.0.10:2377
docker exec worker2 docker swarm join --token $TOKEN 172.30.0.10:2377
docker exec manager docker node ls                        # 확인: 노드 3개 Ready, manager가 Leader

# 2. service 배포
docker exec manager docker service create --name web --replicas 3 -p 8080:80 traefik/whoami:v1.11
docker exec manager docker service ps web                 # 확인: 노드마다 replica 1개씩
curl localhost:8080                                       # 확인: 반복하면 Hostname이 바뀜
                                                          # 확인: http://localhost:8404 에서 노드 3개 UP

# 3. 롤링 업데이트 (다른 터미널에서 curl을 반복해 두면 요청이 끊기지 않는 걸 볼 수 있음)
docker exec manager docker service update --image traefik/whoami:v1.12 \
  --update-parallelism 1 --update-delay 5s web            # 1개씩, 5초 간격으로 교체
docker exec manager docker service ps web                 # 확인: v1.11은 Shutdown, v1.12가 Running
docker exec manager docker service rollback web           # 확인: 다시 v1.11로 돌아감

# 4. stack 배포 (8080을 같이 쓰므로 web을 먼저 지움)
docker exec manager docker service rm web
docker exec manager docker stack deploy -c /stacks/whoami.yml demo
docker exec manager docker stack services demo            # 확인: demo_whoami 6/6 (max 2 per node)
docker exec manager docker stack ps demo                  # 확인: 노드마다 2개씩
curl localhost:8080                                       # 확인: Hostname 6종류가 섞여 나옴
docker exec manager docker stack rm demo                  # 정리: 서비스와 네트워크가 함께 삭제됨

# 재시작 / 초기화
docker compose down && docker compose up -d               # swarm과 서비스 유지
docker compose down -v                                    # 전부 삭제 (swarm을 처음부터 다시 구성)
```

- Hostname 순서가 A → B → C로 돌지 않고 섞여 나오는 건 정상입니다([두 단계 분산](#두-단계-분산)).
- `docker info`의 Driver가 `overlayfs`로 나오는 것도 정상입니다(containerd snapshotter 사용).

### service, 롤링 업데이트, stack

- **service**: 같은 이미지로 띄우는 컨테이너 묶음입니다. replica 수와 포트를 정해 두면 swarm이 노드에 나눠 배치하고 개수를 유지합니다.
- **롤링 업데이트**: service의 이미지나 설정을 바꾸면 replica를 정해진 개수씩 차례로 교체합니다. 나머지 replica가 계속 응답하므로 서비스가 끊기지 않고, `rollback`으로 이전 설정으로 되돌릴 수 있습니다.
- **stack**: compose 파일 하나에 적은 여러 service와 네트워크, 볼륨을 한 번에 배포하는 단위입니다. 이름이 앞에 붙고(`demo_whoami`), `build:`는 지원하지 않아 이미지가 미리 있어야 합니다.

## 구성 설명

### docker-compose.yml

| 대상 | 설정 | 이유 |
|---|---|---|
| 공통 | `image: docker:29-dind`, `privileged: true` | 컨테이너 하나를 "Docker가 설치된 서버"로 씁니다. 안쪽 dockerd가 네트워크, iptables, VXLAN을 만들려면 privileged가 필요합니다. |
| 공통 | `hostname`, `ipv4_address` 고정 | swarm 노드 이름과 advertise 주소로 쓰입니다. 재시작 후에도 같아야 swarm 상태가 유지됩니다. |
| 공통 | `<node>-data:/var/lib/docker` | 이미지와 swarm 상태를 볼륨에 보관해서 재시작해도 유지됩니다. |
| manager | `healthcheck` | dockerd가 응답하고, swarm 상태면 gossip 포트(7946)까지 열려야 healthy입니다. |
| 공통 | `command: dockerd --host=unix:///var/run/docker.sock` | API를 unix 소켓으로만 엽니다([설계 메모 3](#3-docker-api는-unix-소켓만-연다)). |
| worker | `depends_on: manager (service_healthy)` | manager의 gossip이 준비된 뒤에 시작합니다([설계 메모 2](#2-worker는-manager가-준비된-뒤-시작한다)). |
| lb | `haproxy`, `8080:8080`, `8404:8404` | 노드에 직접 포트를 열지 않고 lb를 입구로 씁니다([설계 메모 1](#1-노드에-직접-포트를-열지-않고-haproxy를-둔다)). |

### haproxy/haproxy.cfg

| 설정 | 이유 |
|---|---|
| `server manager/worker1/worker2 :8080`, `balance roundrobin` | routing mesh 덕분에 어느 노드로 보내도 응답하므로, replica 위치를 몰라도 됩니다. |
| `option httpchk GET /`, `check inter 2s fall 2 rise 2` | 주기적으로 HTTP로 확인해서 죽은 노드는 빼고, 살아나면 다시 넣습니다. |
| `retries 2`, `option redispatch`, `timeout connect 2s` | 연결에 실패하면 다른 노드로 재시도합니다. |
| `http-reuse never` | 백엔드 연결을 재사용하면 특정 replica로 쏠릴 수 있어서 끕니다. |

## 핵심 개념: routing mesh

`-p 8080:80`으로 서비스를 만들면 **모든 노드**가 8080을 엽니다. 노드의 8080 뒤에는 IPVS 로드밸런서(`ingress_sbox`)가 있고, 클러스터 **전체** replica 중에서 순서대로 고릅니다. 자기 노드의 replica를 우선하지 않습니다.

```
worker1:8080 ─▶ IPVS ─┬─▶ manager의 replica   (VXLAN으로 이동)
                      ├─▶ worker1의 replica   (로컬)
                      └─▶ worker2의 replica   (VXLAN으로 이동)
```

- 요청을 나누는 건 모든 노드가 똑같이 합니다. manager의 역할은 관리입니다.
- 노드 간 이동은 비효율이지만, 외부 LB가 replica 위치를 몰라도 되는 장점이 있습니다. replica가 옮겨가거나 개수가 바뀌어도 LB 설정은 그대로입니다. [Docker 공식 문서](https://docs.docker.com/engine/swarm/ingress/)도 이 구성(HAProxy + 노드 8080)을 예시로 듭니다.
- replica가 없는 노드로 요청해도 다른 노드의 replica가 응답합니다(`scale web=2` 후 manager:8080으로 요청해 보면 확인 가능).

### 두 단계 분산

HAProxy가 노드를 고르고, 그 노드의 IPVS가 replica를 다시 고릅니다. IPVS는 노드마다 순번을 따로 셉니다(health check도 순번을 가져갑니다). 그래서 순서는 섞이지만 몫은 고르게 나뉩니다.

### 참고: host 모드

`--publish mode=host,target=80,published=8080`을 쓰면 노드의 8080이 **자기 노드의 컨테이너에 직결**되어 노드 간 이동이 없습니다. 대신 한 노드에 replica를 하나만 둘 수 있고(보통 `--mode global`), replica가 없는 노드는 응답하지 않아서 외부 LB health check가 필수입니다. 이 실습에서는 쓰지 않았습니다.

## 설계 메모

### 1. 노드에 직접 포트를 열지 않고 HAProxy를 둔다

manager에 `8080:8080`을 직접 매핑하면 Mac의 `curl`이 **3번 중 1번만 성공**합니다. manager에 있는 replica로 갈 때만 응답하고, 나머지는 응답 없이 멈춥니다.

- **원인:** Docker Desktop 포트포워딩을 거친 패킷의 TCP 체크섬이 `0x0000`입니다. 같은 노드에서 처리하면 문제없지만, VXLAN으로 다른 노드에 넘기면 수신 노드가 체크섬 오류로 버립니다.
  ```
  192.168.65.1.46611 > 172.30.0.10.8080: Flags [S], cksum 0x0000 (incorrect -> 0x5241)
  ```
- **해결:** lb가 연결을 받아 노드로 **새 TCP 연결**을 맺습니다. 새 패킷은 정상 체크섬을 갖습니다.
- 컨테이너 안에서 `ethtool`로 offload를 끄거나 iptables `CHECKSUM` 규칙을 넣는 방법은 효과가 없었습니다.
- Docker Desktop 환경의 문제로, 리눅스 Docker Engine에서는 생기지 않을 가능성이 높습니다.

### 2. worker는 manager가 준비된 뒤 시작한다

세 노드를 동시에 띄우면, worker가 manager의 gossip 포트(7946)가 열리기 직전에 접속을 시도해 실패합니다(`connection refused`). 재시도 주기가 길어서 그동안 manager는 worker의 replica를 몰라 요청이 manager replica로만 갑니다. manager `healthcheck`와 worker `depends_on`으로 순서를 보장합니다.

### 3. Docker API는 unix 소켓만 연다

dind 이미지는 `DOCKER_TLS_CERTDIR=""`이면 **TLS 없는 2375**를 자동으로 여는데, 이때 dockerd가 경고를 보여주려고 시작을 **일부러 늦춥니다**(`Startup is intentionally being slowed down`). 이 실습은 `docker exec`로만 조작하므로 `command`로 unix 소켓만 엽니다.

재시작 시간을 더 줄이려면 서비스에 `--restart-delay 1s`를 주면 됩니다.

## 노드 장애 실습

```bash
while true; do curl -s -m 3 localhost:8080 | grep Hostname || echo FAIL; sleep 0.5; done   # 터미널 1
docker stop manager      # 터미널 2 → 통계 페이지에서 manager DOWN
docker start manager
```

- manager가 멈춰도 요청은 끊기지 않고 나머지 두 replica가 응답합니다.
- manager가 1대라서 멈춘 동안에는 `service create/scale` 같은 관리 작업이 안 됩니다. 관리 기능까지 유지하려면 manager를 3대 이상 홀수로 둬야 합니다.

## 선택: manager 원격 접속 (TLS)

기본 구성에서는 필요 없습니다. `docker exec` 없이 Mac이나 다른 컨테이너(향후 CI)에서 manager를 조작해야 할 때만 켭니다.

### 켜는 방법

`docker-compose.yml`의 manager를 다음처럼 바꿉니다. `command`를 빼면 dind 기본 동작으로 unix 소켓과 TLS 2376을 함께 엽니다.

```yaml
  manager:
    # command: [...]                           → 삭제
    environment:
      DOCKER_TLS_CERTDIR: /certs               # 인증서 자동 생성
    ports:
      - "127.0.0.1:2376:2376"                  # Mac 로컬에만 공개
    volumes:
      - ./stacks:/stacks
      - manager-data:/var/lib/docker
      - manager-certs:/certs                   # 개인키 보관 (재생성해도 같은 키)
      - ./certs/manager-client:/certs/client   # 클라이언트 인증서만 Mac으로 꺼냄

volumes:
  manager-certs:                               # 추가
```

`certs/`는 `.gitignore`에 포함되어 있습니다.

### 인증서

manager가 시작될 때 dind entrypoint가 **컨테이너 안에서** 사설 CA와 인증서를 자동으로 만듭니다. Mac에서 만드는 게 아닙니다.

```
manager:/certs  (manager-certs 볼륨)
  ├── ca/      CA 개인키 (밖으로 나오지 않음)
  ├── server/  dockerd용
  └── client/  ══ bind mount ══▶ Mac: certs/manager-client/{ca,cert,key}.pem
```

- 개인키는 있으면 재사용하고, 인증서만 시작할 때마다 다시 서명합니다. 그래서 컨테이너를 새로 만들어도 기존 클라이언트 인증서가 계속 유효합니다.
- `down -v`를 하면 CA가 새로 만들어집니다. `certs/manager-client`는 자동 갱신되지만, 다른 곳에 복사해 둔 인증서는 다시 복사해야 합니다.
- 서버 인증서 SAN: `manager`, `172.30.0.10`, `localhost`, `127.0.0.1`, `docker`
- `key.pem`이 있으면 manager를 root 권한으로 조작할 수 있습니다. 커밋하지 마세요(`.gitignore` 포함).

### 사용법

`docker exec manager docker ...`는 인증서가 필요 없습니다. TCP로 접속할 때만 필요합니다.

```bash
C=$PWD/certs/manager-client

# ① 명령 플래그
docker -H tcp://127.0.0.1:2376 --tlsverify --tlscacert $C/ca.pem --tlscert $C/cert.pem --tlskey $C/key.pem node ls

# ② 환경변수 (그 터미널의 모든 docker 명령이 manager로 감, 끝나면 unset)
export DOCKER_HOST=tcp://127.0.0.1:2376 DOCKER_TLS_VERIFY=1 DOCKER_CERT_PATH=$C
docker node ls
unset DOCKER_HOST DOCKER_TLS_VERIFY DOCKER_CERT_PATH

# ③ docker context (Mac의 ~/.docker에 저장됨)
docker context create swarm-http --docker "host=tcp://127.0.0.1:2376,ca=$C/ca.pem,cert=$C/cert.pem,key=$C/key.pem"
docker --context swarm-http node ls

# ④ 다른 컨테이너 (CI 형태)
docker run --rm --network swarm-http_swarm-net -v $C:/certs/client:ro \
  -e DOCKER_HOST=tcp://manager:2376 -e DOCKER_TLS_VERIFY=1 -e DOCKER_CERT_PATH=/certs/client \
  docker:29-cli docker service ls
```

Harbor 같은 레지스트리를 붙일 때 노드가 레지스트리를 신뢰하는 데 쓰는 인증서는 이 API 인증서와 **별개**입니다.

## 디버깅 명령

```bash
docker exec manager apk add -q ipvsadm tcpdump                                        # 도구 설치 (재생성 시 사라짐)
docker exec manager nsenter --net=/var/run/docker/netns/ingress_sbox ipvsadm -Ln      # IPVS에 등록된 replica
docker exec manager tcpdump -vvni eth0 udp port 4789                                  # VXLAN 트래픽, 체크섬
docker logs worker1 2>&1 | grep -i gossip                                             # gossip join 실패 여부
curl -s 'localhost:8404/;csv' | awk -F, '$1=="swarm_nodes"{print $2, $18}'            # HAProxy가 보는 노드 상태
docker exec lb wget -qO- http://172.30.0.10:8080                                      # 특정 노드로 직접 요청
```
