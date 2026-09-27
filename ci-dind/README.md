# ci-dind: TLS로 dind 사용하기 (사설 인증서, 내부 전용)

CI job 컨테이너(`ci`)가 별도의 dind 컨테이너(`dind`)에 있는 dockerd로 `docker build`/`run`을 하는 구성입니다. **공인 인증서 없이**, dind가 시작할 때 자동으로 만드는 사설 CA 인증서로 TLS를 켭니다.

```
┌─ ci (docker:29-cli) ─┐                 ┌─ dind (docker:29-dind) ─────────┐
│ DOCKER_HOST=         │   TLS (2376)    │ dockerd --tlsverify             │
│  tcp://docker:2376   │────────────────▶│                                 │
│ /certs/client (ro) ◀─┼── 볼륨 공유 ────┼─ /certs/client (자동 생성)       │
└──────────────────────┘                 └─────────────────────────────────┘
          ci-net (compose 내부 네트워크, 호스트에 포트 노출 없음)
```

## 왜 TLS인가

CLI와 dockerd가 **다른 컨테이너**에 있으면 둘을 잇는 통로는 TCP뿐입니다. 이때 TLS를 끄면(`DOCKER_TLS_CERTDIR=""`, 2375) 문제가 생깁니다.

- 그 포트에 닿는 누구나 root 권한을 얻습니다.
- dockerd가 경고를 보여주려고 시작을 **약 15초** 늦춥니다. [swarm-http README](../swarm-http/README.md)의 트러블슈팅 2 참고.
- 로그에 "향후 버전에서는 hard failure가 된다"는 경고가 나옵니다.

## 사용법

```bash
cd ci-dind
docker compose up -d                               # dind가 healthy가 된 뒤 ci 시작

docker compose exec ci docker version              # TLS로 dind에 접속
docker compose exec ci docker build -t ci-test .   # app/Dockerfile 빌드 (/workspace = ./app)
docker compose exec ci docker run --rm ci-test

docker compose down -v                             # 정리 (인증서, 이미지 포함 삭제)
```

## 구성 요소

| 항목 | 설명 |
|---|---|
| `dind` / `DOCKER_TLS_CERTDIR` 미지정 | 이미지 기본값 `/certs`를 씁니다. 시작할 때 `/certs/{ca,server,client}`를 자동으로 만들고, dockerd를 `--host=tcp://0.0.0.0:2376 --tlsverify`로 띄웁니다. |
| `dind` / `aliases: [docker]` | **필수.** 자동 생성된 서버 인증서에는 `docker`, `localhost`, 컨테이너 ID만 들어 있습니다. 서비스 이름 `dind`로 접속하면 검증에 실패하므로, 인증서에 있는 이름(`docker`)을 네트워크 별칭으로 줍니다. |
| `dind-certs-client` 볼륨 | `/certs/client`(ca.pem, cert.pem, key.pem)만 `ci`와 공유합니다. `ci`는 읽기 전용으로 마운트합니다. CA 개인키가 있는 `/certs/ca`는 공유하지 않습니다. |
| `dind-data` 볼륨 | dind 안의 이미지와 빌드 캐시를 보관합니다. |
| `dind` / `healthcheck` | dind 컨테이너 안에서 unix 소켓으로 `docker info`를 실행해 준비 여부를 확인합니다. |
| `ci` / `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, `DOCKER_CERT_PATH` | 각각 `tcp://docker:2376`, 서버 인증서 검증 켜기, 클라이언트 인증서 위치입니다. |
| `ports` 없음 | 호스트(Mac)에 포트를 노출하지 않습니다. compose 내부 네트워크(`ci-net`)에서만 접근할 수 있습니다. |

## 검증 결과

| 확인 항목 | 결과 |
|---|---|
| dockerd 실행 인자 | `--host=tcp://0.0.0.0:2376 --tlsverify --tlscacert ... --tlscert ... --tlskey ...` |
| 15초 시작 지연 | 없음 (`intentionally being slowed` 로그 0건, `up`부터 healthy까지 3초) |
| 서버 인증서 SAN | `DNS:<컨테이너ID>, DNS:docker, DNS:localhost, IP:127.0.0.1, IP:<컨테이너IP>, IP:::1` |
| `tcp://dind:2376`으로 접속 (별칭 추가 전) | **실패**: `x509: certificate is valid for ..., docker, localhost, not dind` |
| `ci`에서 `docker version` | 성공 (client 29.8.1 / server 29.8.1) |
| `ci`에서 `docker build` + `docker run` | 성공: `built inside dind over TLS` 출력 |

같은 네트워크에 있는 **다른 컨테이너**에서 직접 접속해 봤습니다(curl).

| 시도 | 결과 |
|---|---|
| 클라이언트 인증서 없이 `https://docker:2376` | **거부** (TLS 핸드셰이크 실패, curl exit 56) |
| 다른 사설 CA로 만든 클라이언트 인증서 | **거부** (curl exit 56) |
| 평문 HTTP로 2376 | **거부**: `Client sent an HTTP request to an HTTPS server.` |
| TLS 없는 2375 | 접속 불가 (포트가 열려 있지 않음) |
| 공유된 진짜 클라이언트 인증서 (대조군) | 성공 (`/version` 응답) |
| 호스트 포트 매핑 (`docker port`) | 없음 |

`docker compose ps`의 PORTS에 `2375-2376/tcp`가 보이지만, 이미지에 선언된 `EXPOSE` 정보일 뿐이고 호스트에 publish된 것은 아닙니다.
