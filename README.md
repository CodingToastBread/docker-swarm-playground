# swarm-lab

맥북(Docker Desktop) 한 대에서 Docker-in-Docker(DinD)로 Docker Swarm과 CI/CD 구성 요소를 실습하는 저장소입니다. 실습마다 디렉터리가 나뉘어 있고, 각 디렉터리는 독립된 compose 프로젝트입니다.

| 디렉터리 | 내용 |
|---|---|
| [`swarm-http/`](swarm-http/README.md) | DinD 노드 3개(manager 1 + worker 2)로 swarm을 만들고, HAProxy를 앞에 둬서 HTTP 서비스를 분산합니다. routing mesh 개념, VXLAN 체크섬 문제, 재시작 지연 해결, 노드 장애 실습, manager Docker API의 TLS 원격 접속을 다룹니다. |
| [`ci-dind/`](ci-dind/README.md) | CI job 컨테이너가 별도 dind 컨테이너의 dockerd를 TLS(자동 생성 사설 인증서)로 사용하는 구성입니다. `docker build`/`run`과 인증서 없는 접속 거부를 검증했습니다. |

```
swarm-lab/
├── swarm-http/
│   ├── docker-compose.yml
│   ├── haproxy/haproxy.cfg
│   ├── stacks/whoami.yml
│   └── README.md
├── ci-dind/
│   ├── docker-compose.yml
│   ├── app/Dockerfile
│   └── README.md
└── README.md
```

각 실습은 해당 디렉터리로 이동해서 실행합니다.

```bash
cd swarm-http && docker compose up -d
cd ci-dind && docker compose up -d
```

## 테스트 환경

| 항목 | 버전 |
|---|---|
| macOS | 27.0 (arm64) |
| Docker Desktop 엔진 | 29.8.0 |
| Docker Compose | v5.5.1 |
| dind 이미지 | `docker:29-dind` (29.8.1) |
