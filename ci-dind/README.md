# ci-dind

CI job 컨테이너(`ci`)가 별도 dind 컨테이너(`dind`)의 dockerd로 `docker build`/`run`을 하는 구성입니다. dind가 자동으로 만드는 **사설 CA 인증서로 TLS**를 켜서, 인증서를 가진 컨테이너만 접속할 수 있습니다(인증서가 없거나 다른 CA의 인증서면 거부).

```
ci (docker:29-cli) ── tcp://docker:2376 (TLS) ──▶ dind (docker:29-dind)
   /certs/client (ro) ◀──── dind-certs-client 볼륨 ──── /certs/client (자동 생성)
                    ci-net (내부 네트워크, 호스트 포트 노출 없음)
```

CLI와 dockerd가 다른 컨테이너라 TCP가 필요합니다. TLS 없이(`DOCKER_TLS_CERTDIR=""`, 2375) 열면 접속하는 누구나 root 권한을 얻고, dockerd 시작도 늦어집니다.

## 사용법

```bash
cd ci-dind
docker compose up -d
docker compose exec ci docker version              # TLS로 dind에 접속
docker compose exec ci docker build -t ci-test .   # ./app/Dockerfile 빌드
docker compose exec ci docker run --rm ci-test     # "built inside dind over TLS"
docker compose down -v                             # 정리
```

## 구성 설명

| 대상 | 설정 | 이유 |
|---|---|---|
| dind | `DOCKER_TLS_CERTDIR` 미지정 (기본값 `/certs`) | 시작할 때 인증서를 자동으로 만들고 dockerd를 TLS 2376으로 띄웁니다. |
| dind | 네트워크 별칭 `docker` | **필수.** 서버 인증서 SAN에는 `docker`, `localhost`, 컨테이너 ID만 있어서 서비스 이름 `dind`로 접속하면 검증에 실패합니다(`x509: ... not dind`). |
| dind | `healthcheck: docker info` | ci가 dind 준비 후에 시작하도록 합니다. |
| 볼륨 | `dind-certs-client` | 클라이언트 인증서만 ci와 공유합니다(ci는 읽기 전용). CA 개인키는 공유하지 않습니다. |
| ci | `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, `DOCKER_CERT_PATH` | 미리 설정해 두어서 ci 안에서는 플래그 없이 `docker` 명령을 씁니다. CI 도구에서도 같은 세 변수를 설정합니다. |

## 인증서

- dind 컨테이너 **안에서** 자동으로 만들어집니다(Mac에서 만드는 게 아님).
- 클라이언트 인증서는 named volume에 있어서 Mac의 `ci-dind/`에는 `.pem` 파일이 보이지 않습니다. 확인하거나 꺼내려면 다음 명령을 씁니다.
  ```bash
  docker compose exec ci ls /certs/client
  docker compose cp ci:/certs/client ./certs-copy   # 커밋 금지
  ```
- CA를 볼륨에 보관하지 않아서 dind를 새로 만들면 CA도 바뀝니다. 공유 볼륨의 인증서는 자동으로 갱신되어 ci는 그대로 동작합니다. 인증서를 compose 밖에 복사해 두었다면 다시 복사해야 합니다.
