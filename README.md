# OTA-CE + Aktualizr 설정 가이드

Jetson Orin Nano 보드에서 aktualizr를 실행하고, PC(Mac)에서 ota-ce 서버와 OTA 통신하는 전체 설정 가이드입니다.

---

## 전체 아키텍처

```
[PC / Mac]                          [Jetson Orin Nano]
┌─────────────────────────────┐     ┌─────────────────────────┐
│  ota-ce (Docker)            │     │  aktualizr (client)     │
│  ├── director.ota.ce        │◄────►  config.toml            │
│  ├── reposerver.ota.ce      │mTLS │  └── 인증서 + CA 포함   │
│  ├── treehub.ota.ce         │     └─────────────────────────┘
│  ├── deviceregistry.ota.ce  │
│  └── campaigner.ota.ce      │
└─────────────────────────────┘
```

---

## 사전 요구사항

| 항목 | 버전 |
|------|------|
| Docker Desktop | 최신 버전 |
| Docker Compose | v2 이상 |
| Git | - |
| Jetson OS | JetPack 기반 Ubuntu |

---

## 1단계: PC에서 ota-ce 서버 셋업

### 1-1. 저장소 클론

```bash
git clone https://github.com/SOR-OTA-Team/sor_ota_ce.git
cd sor_ota_ce
```

### 1-2. `/etc/hosts` 설정

`/etc/hosts` 파일에 다음을 추가합니다 (sudo 필요):

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

### 1-3. 서버 인증서 생성

```bash
bash scripts/gen-server-certs.sh
```

### 1-4. 서버 시작

```bash
# DB 먼저 초기화 (20초 대기 권장)
docker-compose -f ota-ce.yaml up db -d
sleep 20

# 전체 서비스 시작
docker-compose -f ota-ce.yaml up -d

# 동작 확인
curl director.ota.ce/health/version
```

### 주요 서비스 포트

| 서비스 | URL | 용도 |
|--------|-----|------|
| Director | `http://director.ota.ce` | 업데이트 지시 |
| Repo Server | `http://reposerver.ota.ce` | TUF 메타데이터 |
| Treehub | `http://treehub.ota.ce` | OSTree 저장소 |
| Device Registry | `http://deviceregistry.ota.ce` | 디바이스 관리 |
| Campaigner | `http://campaigner.ota.ce` | 캠페인 관리 |
| Gateway (mTLS) | `https://ota.ce:30443` | 차량 통신 |
| Traefik Dashboard | `http://localhost:8080` | 관리 UI |

---

## 2단계: 디바이스 인증서 생성 (PC에서)

```bash
bash scripts/gen-device.sh
bash scripts/get-credentials.sh
```

생성되는 파일:
- `ota-ce-gen/devices/<uuid>/config.toml` — aktualizr 설정 파일
- `ota-ce-gen/devices/<uuid>/` — 인증서 및 키 파일
- `credentials.zip` — OTA CLI 사용 시 필요

---

## 3단계: 모니터 없이 Jetson 연결 (헤드리스)

Jetson Orin Nano를 모니터 없이 Mac에서 접속하는 방법입니다.

### 방법 1: USB 시리얼 콘솔 (초기 부팅 확인용, 가장 확실)

```bash
# Mac에서 장치 확인
ls /dev/tty.usbmodem*   # 또는 tty.usbserial*

# 시리얼 접속 (115200 baud)
screen /dev/tty.usbmodem* 115200
```

> Jetson 보드의 USB-C 디버그 포트에 연결

### 방법 2: SSH (네트워크 연결 후)

```bash
ssh <username>@<jetson-ip>
```

> Jetson IP는 라우터 관리 페이지 또는 `arp -a` 명령으로 확인

### 방법 3: 이더넷 직접 연결 (Mac ↔ Jetson)

1. Mac **시스템 환경설정 → 공유 → 인터넷 공유** 활성화
2. 이더넷 케이블로 Mac과 Jetson 직접 연결
3. `arp -a`로 Jetson IP 확인 후 SSH 접속

### 방법 4: USB 가상 이더넷

일부 Jetson 보드는 USB-C 연결 시 가상 네트워크(`usb0`)로 인식됩니다:

```bash
ssh <username>@192.168.55.1
```

---

## 4단계: Jetson에 aktualizr 설치

### 4-1. 의존성 설치

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
  cmake libcurl4-openssl-dev libssl-dev \
  libboost-all-dev libarchive-dev libsodium-dev \
  git build-essential
```

### 4-2. sor_aktualizr 빌드

```bash
git clone https://github.com/SOR-OTA-Team/sor_aktualizr.git
cd sor_aktualizr
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

---

## 5단계: 인증서 복사 및 네트워크 설정

### 5-1. PC → Jetson 인증서 전송

PC에서 실행:

```bash
# <jetson-ip>를 실제 Jetson IP 주소로 변경
scp -r ota-ce-gen/devices/<uuid>/ <username>@<jetson-ip>:~/ota-config/
```

### 5-2. Jetson `/etc/hosts` 설정

Jetson에서 PC의 IP 주소로 도메인 매핑:

```bash
# <PC_IP>를 PC의 실제 IP 주소로 변경
echo "<PC_IP>    ota.ce director.ota.ce reposerver.ota.ce treehub.ota.ce deviceregistry.ota.ce campaigner.ota.ce" | sudo tee -a /etc/hosts
```

---

## 6단계: aktualizr 실행 및 통신 확인

```bash
aktualizr -c ~/ota-config/config.toml
```

정상 동작 시 aktualizr가 director에 접속하여 업데이트를 폴링합니다.

---

## 전체 설정 순서 요약

| 순서 | 위치 | 작업 |
|------|------|------|
| 1 | PC | `sor_ota_ce` 클론 + `/etc/hosts` 설정 |
| 2 | PC | 서버 인증서 생성 (`gen-server-certs.sh`) |
| 3 | PC | Docker 서버 시작 |
| 4 | PC | 디바이스 인증서 생성 (`gen-device.sh`) |
| 5 | Jetson | USB 시리얼 또는 SSH로 접속 |
| 6 | Jetson | `sor_aktualizr` 빌드 및 설치 |
| 7 | Jetson | `/etc/hosts`에 PC IP 등록 |
| 8 | PC → Jetson | `scp`로 인증서/config.toml 복사 |
| 9 | Jetson | `aktualizr -c config.toml` 실행 |

---

## 서버 중지 방법

```bash
# 컨테이너 중지 (데이터 유지)
docker-compose -f ota-ce.yaml down

# 완전 초기화 (데이터 삭제)
docker-compose -f ota-ce.yaml down -v
```

---

## 참고

- [SOR OTA CE Repository](https://github.com/SOR-OTA-Team/sor_ota_ce)
- [SOR Aktualizr Repository](https://github.com/SOR-OTA-Team/sor_aktualizr)
- [Uptane Standard](https://uptane.github.io/)
- [aktualizr 공식 문서](https://github.com/uptane/aktualizr)
