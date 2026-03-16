# SOR OTA Community Edition

> Uptane 기반 차량 OTA(Over-The-Air) 업데이트 플랫폼  
> 원본: [uptane/ota-community-edition](https://github.com/uptane/ota-community-edition)

마이크로서비스 없이 **docker-compose 하나**로 전체 OTA 서버를 로컬에서 실행할 수 있습니다.  
Mac (Apple Silicon / Intel), Windows 모두 지원합니다.

---

## 아키텍처 개요

### 트래픽 흐름

```
[차량/디바이스]                    [개발자/관리자]
      │                                  │
      │ mTLS 필수                        │ 일반 HTTP/HTTPS
      ▼                                  ▼
gateway:30443                    reverse-proxy:80
      │                                  │
      │ 버전 확인, 메타데이터             │ 이미지 업로드
      │ 업데이트 다운로드                 │ 디바이스 관리
      │ 상태 보고                        │ 캠페인 생성
      ▼                                  ▼
   ota-lith                          ota-lith
```

- **gateway(30443)** — 차량 전용, mTLS 인증으로 허가된 차량만 접근 가능
- **reverse-proxy(80)** — 관리자/개발자 전용 (추후 관리자 인증 추가 필요)

### 상세 라우팅

```
[차량]
    │
    ▼
gateway:30443 (mTLS)
    │
    ├─ /treehub/    ─────────→ ota-lith:7400  (OSTree 저장소)
    ├─ /repo/       ─────────→ ota-lith:7100  (TUF 메타데이터)
    ├─ /campaigner/ ─────────→ ota-lith:7600  (캠페인)
    ├─ /core/       ─────────→ ota-lith:7500  (코어 서비스)
    └─ /director/   ─→ reverse-proxy:80 ─→ ota-lith (director)

[개발자]
    │
    ▼
reverse-proxy:80 → ota-lith (관리 API)
```

> `/director/` 경로만 reverse-proxy를 경유하고, 나머지는 gateway에서 ota-lith로 직접 연결됩니다.  
> 현재는 포트 번호로 직접 연결되며, 추후 Traefik의 서비스 디스커버리로 서비스 추가/변경 시 gateway 설정을 건드리지 않도록 개선 예정입니다.

### 포트 정리

| 컨테이너 | 포트 | 용도 | 보안 |
|---|---|---|---|
| gateway | 30443 | HTTPS (mTLS) — 차량 전용 | ✅ mTLS |
| reverse-proxy | 80 | HTTP — 관리자/개발자 | ⚠️ 인증 없음 |
| reverse-proxy | 8080 | Traefik 대시보드 | ⚠️ 인증 없음 |
| db | 3306 | MariaDB | ⚠️ 비밀번호만 |
| kafka | 9092 | Kafka 브로커 | ❌ 인증 없음 |
| zookeeper | 2181 | Kafka 코디네이터 | ❌ 인증 없음 |

> **주의:** 아래 포트들은 현재 보안 설정이 없으므로 운영 환경에서는 방화벽으로 차단하거나 인증을 추가해야 합니다.
> - DB(3306): 비밀번호만 알면 접근 가능
> - Kafka(9092): 인증 없이 메시지 조작 가능
> - Zookeeper(2181): Kafka 클러스터 설정 변경 가능
> - reverse-proxy(80, 8080): mTLS 없는 평문 HTTP

---

## 사전 요구사항

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) 설치 및 실행
- 메모리: Docker Desktop → Settings → Resources → **4GB 이상** 설정

---

## 최초 1회 설정

### 1. 저장소 클론

```bash
git clone https://github.com/seongyongju/sor_ota.git
cd sor_ota
```

### 2. /etc/hosts 설정

**Mac / Linux**
```bash
sudo nano /etc/hosts
```

아래 내용 추가:
```
127.0.0.1    reposerver.ota.ce
127.0.0.1    keyserver.ota.ce
127.0.0.1    director.ota.ce
127.0.0.1    treehub.ota.ce
127.0.0.1    deviceregistry.ota.ce
127.0.0.1    campaigner.ota.ce
127.0.0.1    app.ota.ce
127.0.0.1    ota.ce
```

> `0.0.0.0` 대신 `127.0.0.1`을 사용합니다. 둘 다 동작하지만 `127.0.0.1`이 로컬 루프백 주소를 명시하여 더 안전하고 명확합니다.

**Windows**  
`C:\Windows\System32\drivers\etc\hosts` 파일을 **메모장(관리자 권한)** 으로 열고 아래 내용 추가

```
127.0.0.1    reposerver.ota.ce
127.0.0.1    keyserver.ota.ce
127.0.0.1    director.ota.ce
127.0.0.1    treehub.ota.ce
127.0.0.1    deviceregistry.ota.ce
127.0.0.1    campaigner.ota.ce
127.0.0.1    app.ota.ce
127.0.0.1    ota.ce
```

### 3. 서버 인증서 생성

**Mac / Linux**
```bash
bash scripts/gen-server-certs.sh
```

**Windows (Git Bash)**

Windows에서는 스크립트 실행 전 줄바꿈 문자 변환이 필요합니다.
```bash
dos2unix scripts/gen-server-certs.sh
bash scripts/gen-server-certs.sh
```

> Windows는 줄바꿈 문자가 `\r\n`(CRLF)이라 bash 스크립트 실행 시 아래와 같은 오류가 발생합니다.
> ```
> gen-server-certs.sh: line 2: $'\r': command not found
> : invalid option nameline 3: set: pipefail
> ```
> `dos2unix`로 `\n`(LF)으로 변환 후 실행하면 해결됩니다.

---

## 실행

```bash
# DB 먼저 기동 (10~20초 대기)
docker-compose -f ota-ce.yaml up db -d

# 전체 서비스 기동
docker-compose -f ota-ce.yaml up -d

# 상태 확인
docker-compose -f ota-ce.yaml ps
```

모든 서비스가 `Up` 상태이면 정상입니다.

```bash
# 동작 확인
curl director.ota.ce/health/version
```

---

## 접속 주소

| 용도 | 주소 |
|---|---|
| Director | http://director.ota.ce |
| Device Registry | http://deviceregistry.ota.ce |
| Repo Server | http://reposerver.ota.ce |
| Treehub | http://treehub.ota.ce |
| Traefik 대시보드 | http://localhost:8080 |

---

## 디바이스 등록 및 업데이트 배포

```bash
# 새 디바이스 인증서 생성
bash scripts/gen-device.sh
# → ota-ce-gen/devices/<uuid>/ 에 설정 파일 생성됨

# credentials.zip 생성 (API/CLI 사용 시 필요)
bash scripts/get-credentials.sh
```

업데이트 배포 방법:
- API 사용 → [`docs/api-updates.md`](docs/api-updates.md)
- ota-cli 사용 → [`docs/updates-ota-cli.md`](docs/updates-ota-cli.md)

---

## 종료

```bash
# 컨테이너 종료 (데이터 유지)
docker-compose -f ota-ce.yaml down

# 컨테이너 + 데이터 전체 삭제 (초기화)
docker-compose -f ota-ce.yaml down -v
```

---

## 사용 이미지

모든 이미지는 `linux/amd64` (Windows, Intel Mac), `linux/arm64` (Apple Silicon) 멀티 플랫폼을 지원합니다.

| 서비스 | 이미지 |
|---|---|
| DB | `jusy4901/ota-ce-db:latest` (MariaDB 10.4) |
| Gateway | `jusy4901/ota-ce-gateway:latest` (Nginx) |
| Kafka | `jusy4901/ota-ce-kafka:latest` |
| OTA 서버 | `jusy4901/ota-ce-lith:latest` |
| Reverse Proxy | `jusy4901/ota-ce-proxy:latest` (Traefik) |
| Zookeeper | `jusy4901/ota-ce-zookeeper:latest` |

---

## 자주 묻는 질문

**Q. `Got timeout reading communication packets` 로그가 계속 뜹니다.**  
A. 정상입니다. 재시작 시 이전 DB 연결이 정리되는 과정에서 나오는 Warning이며 동작에 영향 없습니다.

**Q. `docker-compose: command not found` 에러가 납니다.**  
A. 최신 Docker Desktop은 띄어쓰기 버전을 사용합니다.
```bash
docker compose -f ota-ce.yaml up -d
```

**Q. Windows에서 포트 충돌 에러가 납니다.**  
A. 3306, 80, 9092, 2181 포트가 이미 사용 중인지 확인하세요.
```powershell
netstat -ano | findstr :3306
```

**Q. Windows에서 `$'\r': command not found` 에러가 납니다.**  
A. Windows의 줄바꿈 문자(`\r\n`) 때문입니다. `dos2unix`로 변환 후 실행하세요.
```bash
dos2unix scripts/gen-server-certs.sh
dos2unix scripts/gen-device.sh
bash scripts/gen-server-certs.sh
```

**Q. `gateway` 컨테이너가 재시작을 반복합니다.**  
A. 대부분 `/etc/hosts` 설정이 누락되어 있거나 인증서가 없는 경우입니다. 최초 1회 설정의 2~3번 단계를 완료했는지 확인하세요.

**Q. `Could not resolve host: director.ota.ce` 에러가 납니다.**  
A. `/etc/hosts` 설정이 적용되지 않은 것입니다. 터미널을 재시작하거나 2번 단계를 다시 수행하세요.
