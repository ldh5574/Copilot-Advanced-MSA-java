# 폐쇄망 환경 Docker 이미지 사용 가이드

이 폴더에는 폐쇄망(Air-gapped) 환경에서 MSA 프로젝트를 실행하기 위한 Docker 이미지가 저장되어 있습니다.

## 📦 포함된 파일 목록

### 필수 파일

| 파일명 | 용도 | 크기 |
|--------|------|------|
| `registry.tar` | 로컬 Docker 레지스트리 이미지 | ~10MB |
| `registry-data.tar.gz` | 레지스트리 볼륨 데이터 (모든 이미지 포함) | ~884MB |

### 포함된 이미지 (registry-data 내부)

| 이미지 | 태그 | 용도 |
|--------|------|------|
| cp-zookeeper | 7.5.0 | Kafka 의존성 |
| cp-kafka | 7.5.0 | 메시지 브로커 |
| kafka-ui | latest | Kafka 모니터링 UI |
| temurin | 17-jdk-alpine | Java 빌드용 JDK |
| temurin | 17-jre-alpine | Java 런타임 JRE |

---

## 🚀 사용 방법 (권장: 레지스트리 볼륨 방식)

**이 방식은 `docker load`를 사용하지 않습니다.** 레지스트리 데이터를 직접 복사하여 사용합니다.

### Step 1: 레지스트리 이미지 Load 및 데이터 준비

```bash
cd docker-images

# 레지스트리 이미지만 로드 (이것만 docker load 사용)
docker load -i registry.tar

# 레지스트리 데이터 압축 해제
tar -xzf registry-data.tar.gz
```

### Step 2: 로컬 레지스트리 실행 (볼륨 마운트)

**⚠️ 중요: Windows에서는 반드시 절대 경로를 사용해야 합니다!**

**Windows (권장):**
```powershell
# 프로젝트 루트로 이동
cd ..

# 예시:
docker run -d -p 5000:5000 --restart=always --name local-registry `
  -v "C:\Users\사용자명\프로젝트\docker-images\registry-data:/var/lib/registry" `
  registry:2
```

```powershell

# 레지스트리 실행 (데이터 폴더 마운트) Bash 에서실행
 docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name local-registry \
  -v "C:\Users\hjoong.TANGUNSOFT\kube-test\copilot\Copilot-Advanced-MSA-Java\docker-images\registry-data:/var/lib/registry" \
  registry:2
```

> **Windows 주의사항**: `$(pwd)`나 `${PWD}` 사용 시 경로가 잘못 변환되어 볼륨 마운트가 실패할 수 있습니다. 반드시 절대 경로를 직접 입력하세요.

### Step 3: 레지스트리 확인

```bash
# 이미지 목록 확인 (curl 또는 브라우저)
curl http://localhost:5000/v2/_catalog

# 예상 결과: {"repositories":["cp-kafka","cp-zookeeper","kafka-ui","temurin"]}
```

### Step 4: Docker Compose 실행

```bash
# 프로젝트 루트에서 실행
docker compose up --build -d
```

> **참고**: `docker-compose.yml`과 Dockerfile이 이미 `localhost:5000/*` 이미지를 사용하도록 설정되어 있습니다.

---

## 🔧 설정 파일 확인

### docker-compose.yml

```yaml
zookeeper:
  image: localhost:5000/cp-zookeeper:7.5.0
kafka:
  image: localhost:5000/cp-kafka:7.5.0
kafka-ui:
  image: localhost:5000/kafka-ui:latest
```

### Dockerfile (각 서비스)

```dockerfile
FROM localhost:5000/temurin:17-jdk-alpine AS builder
...
FROM localhost:5000/temurin:17-jre-alpine
```

---

## 🔧 Gradle 캐시 사용 방법

### 자동 사용 (권장)

Dockerfile이 이미 Gradle 캐시를 자동으로 사용하도록 설정되어 있습니다:
1. `docker-images/gradle-cache.tar.gz` 파일이 있으면 자동으로 복사
2. 캐시가 있으면 `--offline` 모드로 빌드 시도
3. 오프라인 빌드 실패 시 일반 빌드로 폴백

### 새 캐시 생성 (외부망에서)

의존성이 변경되었거나 캐시를 새로 만들어야 할 경우:

```bash
# 1. 의존성 다운로드
./gradlew build --refresh-dependencies --no-daemon

# 2. 캐시 압축
cd ~/.gradle
tar -czf gradle-cache.tar.gz caches

# 3. docker-images 폴더로 이동
mv gradle-cache.tar.gz /path/to/project/docker-images/
```

---

## 🔧 Windows 환경 주의사항

Windows에서 빌드 시 `gradlew` 파일의 줄바꿈 문제가 발생할 수 있습니다.
현재 Dockerfile에는 이미 다음 명령이 포함되어 있습니다:

```dockerfile
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew
```

---

## ✅ 서비스 확인

모든 서비스가 정상 실행되면 다음 URL로 접근할 수 있습니다:

| 서비스 | URL | 설명 |
|--------|-----|------|
| API Gateway | http://localhost:8080 | 메인 진입점 |
| User Service | http://localhost:8081 | 사용자 서비스 |
| Product Service | http://localhost:8082 | 상품 서비스 |
| Order Service | http://localhost:8083 | 주문 서비스 |
| Kafka UI | http://localhost:8090 | Kafka 모니터링 |

### 상태 확인 명령어

```bash
# 컨테이너 상태 확인
docker compose ps

# 서비스 로그 확인
docker compose logs -f [서비스명]

# API 테스트
curl http://localhost:8080/api/users
curl http://localhost:8080/api/products
curl http://localhost:8080/api/orders
```

---

## 🛑 서비스 중지

```bash
# 서비스 중지
docker compose down

# 로컬 레지스트리 중지
docker stop local-registry
docker rm local-registry
```

## 🛑 로컬 레지스트리, -v 볼륨 구성 실패시, 재시작 명령어
```bash
#1.실행중인 컨테이너 전부 내리기
docker stop $(docker ps -q)
#2.컨테이너 전부 삭제
docker rm -f $(docker ps -aq)
#3.docker run 명령어 재실행
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name local-registry \
  -v "C:\Users\{프로젝트 경로}\Copilot-Advanced-MSA-Java\docker-images\registry-data:/var/lib/registry" \
  registry:2

```

---
## ❓ 문제 해결

### Q: 레지스트리 카탈로그가 비어있음 (`{"repositories":[]}`)
**A:** 두 가지 원인이 있을 수 있습니다:

**1. 볼륨 마운트 경로 문제 (Windows에서 흔함)**
```powershell
# Windows에서는 절대 경로를 직접 사용해야 합니다
docker stop local-registry
docker rm local-registry
docker run -d -p 5000:5000 --restart=always --name local-registry `
  -v "C:\절대\경로\docker-images\registry-data:/var/lib/registry" `
  registry:2
```

**2. 압축 해제가 안 된 경우**
```bash
ls docker-images/registry-data/docker/registry/v2/repositories/
# cp-kafka, cp-zookeeper, kafka-ui, temurin 폴더가 있어야 함
```

### Q: 빌드 시 이미지를 찾을 수 없음
**A:** 로컬 레지스트리가 실행 중인지 확인하세요:
```bash
docker ps | grep registry
curl http://localhost:5000/v2/_catalog
```

### Q: 빌드 시 "gradlew: not found" 오류
**A:** Windows 줄바꿈 문제입니다. Dockerfile에 `sed -i 's/\r$//' gradlew` 명령이 포함되어 있는지 확인하세요.

### Q: 빌드 시 의존성 다운로드 실패 (폐쇄망)
**A:** `gradle-cache.tar.gz` 파일이 `docker-images/` 폴더에 있는지 확인하세요. 없으면 외부망에서 새로 생성해야 합니다.

### Q: 오프라인 빌드 실패
**A:** Gradle 캐시가 불완전할 수 있습니다. 외부망에서 `./gradlew build --refresh-dependencies`를 실행하여 캐시를 다시 생성하세요.

### Q: 서비스가 시작되지 않음
**A:** 로그를 확인하세요: `docker compose logs [서비스명]`

---

## 🔄 대안: 직접 Load 방식

레지스트리 볼륨 방식이 동작하지 않는 경우, 개별 tar 파일을 직접 로드할 수 있습니다.

### 필요한 파일 (별도 준비)
- `zookeeper.tar`
- `kafka.tar`
- `kafka-ui.tar`
- `temurin-jdk.tar`
- `temurin-jre.tar`

### 로드 방법

```bash
cd docker-images

# 이미지 로드
docker load -i zookeeper.tar
docker load -i kafka.tar
docker load -i kafka-ui.tar
docker load -i temurin-jre.tar

# temurin-jdk.tar가 멈추는 경우 docker import 사용
docker import temurin-jdk.tar eclipse-temurin:17-jdk-alpine
```

> **주의**: 이 방식 사용 시 `docker-compose.yml`과 Dockerfile의 이미지 경로를 원본 이미지명으로 변경해야 합니다.

---

## 📁 폴더 구조

```
docker-images/
├── docker-air-gapped-net-guide.md  # 이 문서
├── gradle-cache.tar.gz             # Gradle 의존성 캐시 (~576MB)
├── registry.tar                    # Docker Registry 이미지 (~10MB)
├── registry-data.tar.gz            # 레지스트리 볼륨 데이터 (~884MB)
└── registry-data/                  # 압축 해제된 레지스트리 데이터
    └── docker/
        └── registry/
            └── v2/
                └── repositories/
                    ├── cp-kafka/
                    ├── cp-zookeeper/
                    ├── kafka-ui/
                    └── temurin/
```

**총 용량**: 약 1.5GB (압축 상태)
