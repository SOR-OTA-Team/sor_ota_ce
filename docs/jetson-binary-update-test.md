# Jetson Nano 바이너리 OTA 업데이트 테스트 가이드

Mac에서 OTA-CE 서버를 실행하고 Jetson Nano(또는 다른 Linux 보드)에 aktualizr를 설치하여  
바이너리 파일을 OTA로 전송하는 전체 과정을 설명합니다.

---

## 구성

```
[Mac] OTA-CE 서버 (Docker)
   ├── gateway:30443  ─── mTLS ──→  [Jetson] aktualizr
   └── reverse-proxy:80 (관리 API)
```

---

## 1. Mac — OTA-CE 서버 구동

### 1-1. 서버 인증서 생성 (최초 1회)

```bash
bash scripts/gen-server-certs.sh
```

### 1-2. 서버 실행

```bash
docker compose -f ota-ce.yaml up db -d
sleep 15
docker compose -f ota-ce.yaml up -d
```

### 1-3. 헬스 체크

```bash
curl http://director.ota.ce/health
curl http://reposerver.ota.ce/health
```

모두 `{"status":"OK"}` 응답이 나오면 정상.

### 1-4. credentials.zip 생성

```bash
bash scripts/get-credentials.sh
```

`ota-ce-gen/credentials.zip` 생성됨.

---

## 2. Mac — 디바이스 인증서 생성

Jetson이 게이트웨이(mTLS)에 접속하기 위한 클라이언트 인증서를 생성합니다.

```bash
DEVICE_UUID=$(uuidgen | tr '[:upper:]' '[:lower:]')
echo "Device UUID: $DEVICE_UUID"

openssl genrsa -out ota-ce-gen/device.key 2048
openssl req -new -key ota-ce-gen/device.key -subj "/CN=${DEVICE_UUID}" -out ota-ce-gen/device.csr
openssl x509 -req \
  -in ota-ce-gen/device.csr \
  -CA ota-ce-gen/devices/ca.crt \
  -CAkey ota-ce-gen/devices/ca.key \
  -CAcreateserial \
  -out ota-ce-gen/device.crt \
  -days 36500

# UUID 저장
echo $DEVICE_UUID > ota-ce-gen/device_uuid.txt
```

---

## 3. Jetson — aktualizr 설치 및 설정

### 3-1. Mac에서 Jetson으로 파일 전송

```bash
JETSON_IP="<Jetson IP 주소>"
JETSON_USER="<사용자명>"

scp ota-ce-gen/credentials.zip ${JETSON_USER}@${JETSON_IP}:~/
scp ota-ce-gen/device.key ota-ce-gen/device.crt ota-ce-gen/devices/ca.crt ota-ce-gen/server_ca.pem \
    ${JETSON_USER}@${JETSON_IP}:~/
```

### 3-2. Jetson — /etc/hosts에 Mac IP 추가

Mac IP 확인 (Mac 터미널에서):
```bash
ifconfig | grep "inet " | grep -v 127
```

Jetson에서:
```bash
MAC_IP="<Mac IP 주소>"
sudo bash -c "echo '${MAC_IP}  reposerver.ota.ce keyserver.ota.ce director.ota.ce treehub.ota.ce ota.ce' >> /etc/hosts"
```

헬스 체크:
```bash
curl http://reposerver.ota.ce/health
```

### 3-3. Jetson — 인증서 배치

```bash
sudo mkdir -p /var/sota/import /var/lib/aktualizr

sudo cp ~/device.crt /var/sota/import/client.pem
sudo cp ~/device.key /var/sota/import/pkey.pem
sudo cp ~/server_ca.pem /var/sota/import/root.crt

# 확인
sudo head -1 /var/sota/import/client.pem   # -----BEGIN CERTIFICATE-----
sudo head -1 /var/sota/import/pkey.pem     # -----BEGIN PRIVATE KEY-----
```

### 3-4. Jetson — aktualizr 설정 파일 작성

`DEVICE_UUID`는 앞서 Mac에서 생성한 UUID (device_uuid.txt 참고):

```bash
DEVICE_UUID="<device_uuid.txt 내용>"

sudo vi /etc/aktualizr/aktualizr.toml
```

아래 내용 입력:

```toml
[tls]
server = "https://ota.ce:30443"
ca_source = "file"
pkey_source = "file"
cert_source = "file"

[provision]
provision_path = "/home/<사용자명>/credentials.zip"
device_id = "<DEVICE_UUID>"
primary_ecu_hardware_id = "jetson-nano"
ecu_registration_endpoint = "https://ota.ce:30443/director/ecus"
mode = "DeviceCred"

[uptane]
polling_sec = 10
director_server = "https://ota.ce:30443/director"
repo_server = "https://ota.ce:30443/repo"

[storage]
type = "sqlite"
path = "/var/lib/aktualizr"

[import]
base_path = "/var/sota/import"
tls_cacert_path = "root.crt"
tls_pkey_path = "pkey.pem"
tls_clientcert_path = "client.pem"

[pacman]
type = "none"

[bootloader]
rollback_mode = "none"
```

---

## 4. Mac — 디바이스 등록

aktualizr를 처음 실행하기 전에 디바이스를 device registry에 등록합니다.

```bash
DEVICE_UUID=$(cat ota-ce-gen/device_uuid.txt)

curl -s -X POST "http://deviceregistry.ota.ce/api/v1/devices" \
  -H "x-ats-namespace:default" \
  -H "Content-Type: application/json" \
  -d "{\"deviceName\":\"jetson-nano\",\"deviceId\":\"${DEVICE_UUID}\",\"deviceType\":\"Other\"}"
```

---

## 5. Jetson — aktualizr 최초 실행

```bash
sudo aktualizr --config /etc/aktualizr/aktualizr.toml --loglevel 0
```

정상 연결 시 로그에 다음이 나타납니다:
- `SSL certificate verify ok`
- `Successfully imported client certificate`
- `provisioned OK`

---

## 6. Mac — 바이너리 업로드 및 업데이트 배포

스크립트를 사용하면 업로드부터 배포까지 자동화됩니다:

```bash
bash scripts/push-update.sh <파일경로> <하드웨어ID> <디바이스UUID>
```

예시:
```bash
echo "hello OTA world v1" > test-binary.bin
bash scripts/push-update.sh test-binary.bin jetson-nano $(cat ota-ce-gen/device_uuid.txt)
```

### 수동 진행 방법

#### 6-1. 바이너리 업로드

```bash
FILE=test-binary.bin
SHA256=$(shasum -a 256 $FILE | awk '{print $1}')
SIZE=$(wc -c < $FILE | tr -d ' ')
NAME=$(basename $FILE)

curl -g -X PUT \
  "http://reposerver.ota.ce/api/v1/user_repo/targets/${NAME}?name=${NAME}&version=1.0.0&hardwareIds=jetson-nano&length=${SIZE}&checksum%5Bmethod%5D=sha256&checksum%5Bhash%5D=${SHA256}" \
  -H "x-ats-namespace:default" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @${FILE}
```

#### 6-2. 업데이트 할당 (director DB 직접)

```bash
DEVICE_UUID=$(cat ota-ce-gen/device_uuid.txt)

# ECU 시리얼 조회
ECU_SERIAL=$(docker exec sor_ota_ce-db-1 mysql -uroot -proot director_v2 \
  -se "SELECT ecu_serial FROM ecus WHERE device_id='${DEVICE_UUID}';" 2>/dev/null)

# ecu_targets ID 조회
TARGET_ID=$(docker exec sor_ota_ce-db-1 mysql -uroot -proot director_v2 \
  -se "SELECT id FROM ecu_targets WHERE filename='${NAME}' ORDER BY created_at DESC LIMIT 1;" 2>/dev/null)

# assignment 삽입
CORR_ID="urn:here-ota:campaign:$(uuidgen | tr '[:upper:]' '[:lower:]')"
docker exec sor_ota_ce-db-1 mysql -uroot -proot director_v2 -e "
INSERT IGNORE INTO assignments (namespace, device_id, ecu_serial, ecu_target_id, correlation_id, in_flight)
VALUES ('default', '${DEVICE_UUID}', '${ECU_SERIAL}', '${TARGET_ID}', '${CORR_ID}', 0);" 2>/dev/null

# 메타데이터 재생성 트리거
docker exec sor_ota_ce-db-1 mysql -uroot -proot director_v2 -e \
  "UPDATE devices SET generated_metadata_outdated=1 WHERE id='${DEVICE_UUID}';" 2>/dev/null
```

---

## 7. Jetson — 업데이트 수신 확인

```bash
sudo aktualizr --config /etc/aktualizr/aktualizr.toml --loglevel 0 2>&1 | \
  grep -i "download\|install\|EcuDownload\|EcuInstall" | grep -v "force_install\|preinstall"
```

다음 이벤트가 나오면 성공:
```
"id" : "EcuDownloadStarted"
"id" : "EcuDownloadCompleted"   ← success: true
"id" : "EcuInstallationStarted"
"id" : "EcuInstallationCompleted"  ← success: true
```

### 다운로드된 파일 확인

파일은 SHA256 해시 이름으로 저장됩니다:
```bash
ls /var/sota/images/
cat /var/sota/images/<SHA256_HASH>
```

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| `Couldn't resolve host: ota.ce` | /etc/hosts 설정 오류 | `cat /etc/hosts`로 줄바꿈 없이 한 줄인지 확인 |
| `Client certificate not found` | client.pem/pkey.pem 파일 불일치 | `head -1`로 CERTIFICATE/PRIVATE KEY 확인 |
| `No new updates found` | director targets.json이 갱신 안 됨 | `generated_metadata_outdated=1` 설정 |
| `SSL certificate problem: self-signed` | system_info 리다이렉트 오류 | 무시 가능 (업데이트 수신에는 영향 없음) |
| `missing_device` | device registry 미등록 | 4번 단계 수동 등록 필요 |
| `Invalid correlationId` | correlationId 형식 오류 | `urn:here-ota:campaign:<uuid>` 형식 사용 |
