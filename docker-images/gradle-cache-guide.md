# Gradle 캐시 폐쇄망 설정 가이드

폐쇄망(Air-gapped) 환경에서 Gradle 빌드를 위한 캐시 준비 및 사용 방법을 설명합니다.

---

## 📦 개요

폐쇄망에서는 Gradle이 Maven Central에서 의존성을 다운로드할 수 없습니다.  
따라서 외부망에서 미리 의존성을 다운로드하여 캐시 파일로 만들고, 이를 폐쇄망으로 이관해야 합니다.

### 포함된 파일

| 파일명 | 용도 | 크기 |
|--------|------|------|
| `gradle-cache.tar.gz` | Gradle 의존성 캐시 | ~576MB |

---

## 🔧 외부망에서 캐시 생성 방법

### Step 1: 의존성 다운로드

#### Windows (PowerShell)

```powershell
# 프로젝트 루트에서 실행
# JAVA_HOME 설정 (필요시)
$env:JAVA_HOME = "D:\IntelliJ IDEA 2025.2.4\jbr"
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"

# Gradle 빌드 실행 (모든 의존성 다운로드)
./gradlew build --refresh-dependencies --no-daemon
```

#### Linux/Mac (Bash)

```bash
# 프로젝트 루트에서 실행
# JAVA_HOME 설정 (필요시)
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export PATH=$JAVA_HOME/bin:$PATH

# Gradle 빌드 실행 (모든 의존성 다운로드)
./gradlew build --refresh-dependencies --no-daemon
```

### Step 2: 캐시 압축

```bash
# Windows PowerShell
cd $env:USERPROFILE\.gradle
tar -czf gradle-cache.tar.gz caches

# 프로젝트의 docker-images 폴더로 이동
Move-Item gradle-cache.tar.gz C:\Users\hjoong.TANGUNSOFT\kube-test\copilot\Copilot-Advanced-MSA-Java\docker-images
```

```bash
# Linux/Mac
cd ~/.gradle
tar -czf gradle-cache.tar.gz caches

# 프로젝트의 docker-images 폴더로 이동
mv gradle-cache.tar.gz /path/to/project/docker-images/
```

### Step 3: 확인

```bash
# 파일 크기 확인 (약 500MB 이상이어야 함)
ls -lh docker-images/gradle-cache.tar.gz
```

---

## 📁 Dockerfile 설정

각 서비스의 Dockerfile에 다음 로직이 포함되어 있습니다:

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app

# Gradle 캐시 복사 (폐쇄망용 - 있는 경우에만)
COPY docker-images/gradle-cache.tar.gz* ./
RUN if [ -f gradle-cache.tar.gz ]; then \
        mkdir -p /root/.gradle && \
        tar -xzf gradle-cache.tar.gz -C /root/.gradle && \
        rm gradle-cache.tar.gz; \
    fi

COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .
COPY common common
COPY [service-name] [service-name]
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew

# 오프라인 모드로 빌드 (캐시가 있으면)
RUN if [ -d /root/.gradle/caches ]; then \
        ./gradlew :[service-name]:bootJar --no-daemon --offline || \
        ./gradlew :[service-name]:bootJar --no-daemon; \
    else \
        ./gradlew :[service-name]:bootJar --no-daemon; \
    fi

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/[service-name]/build/libs/*.jar app.jar
EXPOSE [port]
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 주요 포인트

1. **`COPY docker-images/gradle-cache.tar.gz* ./`**
   - `*` 와일드카드로 파일이 없어도 에러가 발생하지 않음
   - 외부망에서는 캐시 없이도 빌드 가능

2. **조건부 캐시 압축 해제**
   - 캐시 파일이 있으면 `/root/.gradle`에 압축 해제
   - 압축 해제 후 tar 파일 삭제 (이미지 크기 최적화)

3. **오프라인 빌드 시도**
   - 캐시가 있으면 `--offline` 모드로 먼저 시도
   - 실패하면 일반 빌드로 폴백 (네트워크 필요)

---

## 🚀 폐쇄망에서 사용 방법

### Step 1: 파일 준비

폐쇄망 PC에 다음 파일들을 복사:
- `docker-images/gradle-cache.tar.gz`
- `docker-images/*.tar` (Docker 이미지들)

### Step 2: Docker 이미지 로드

```bash
cd docker-images
docker load -i temurin-jdk.tar
docker load -i temurin-jre.tar
docker load -i zookeeper.tar
docker load -i kafka.tar
docker load -i kafka-ui.tar
```

### Step 3: 빌드 및 실행

```bash
cd ..  # 프로젝트 루트로 이동
docker compose up --build -d
```

> Dockerfile이 자동으로 `gradle-cache.tar.gz`를 감지하여 오프라인 빌드를 시도합니다.

---

## ⚠️ 주의사항

### 1. 캐시 버전 관리

의존성이 변경되면 캐시를 다시 생성해야 합니다:

```bash
# build.gradle.kts 변경 후
./gradlew build --refresh-dependencies --no-daemon

# 캐시 재생성
cd ~/.gradle
rm -f gradle-cache.tar.gz
tar -czf gradle-cache.tar.gz caches
```

### 2. Gradle Wrapper 다운로드

`gradle-cache.tar.gz`에는 Gradle Wrapper(`gradle-8.5-bin.zip`)가 포함되지 않습니다.  
완전한 오프라인 빌드를 위해서는 Gradle Wrapper도 별도로 준비해야 합니다:

```bash
# Gradle Wrapper 다운로드 위치
~/.gradle/wrapper/dists/gradle-8.5-bin/

# 압축에 포함시키려면
cd ~/.gradle
tar -czf gradle-cache-full.tar.gz caches wrapper
```

### 3. 플랫폼 호환성

- 캐시는 플랫폼 독립적이므로 Windows에서 생성해도 Linux(Docker)에서 사용 가능
- 단, 일부 네이티브 라이브러리는 플랫폼별로 다를 수 있음

---

## 🔍 문제 해결

### Q: 오프라인 빌드 실패

**증상:**
```
> Could not resolve all files for configuration ':common:compileClasspath'.
> Could not find org.springframework.boot:spring-boot-starter-web:3.2.1.
```

**해결:**
1. 캐시 파일이 `docker-images/` 폴더에 있는지 확인
2. 캐시를 새로 생성 (외부망에서 `--refresh-dependencies` 옵션 사용)

### Q: 캐시 파일이 너무 큼

**해결:**
불필요한 캐시 정리 후 재생성:
```bash
# 오래된 캐시 삭제
rm -rf ~/.gradle/caches/modules-2/files-2.1/*/
./gradlew build --no-daemon
```

### Q: 빌드 시 "gradlew: not found" 오류

**해결:**
Windows 줄바꿈 문제입니다. Dockerfile에 다음 명령이 있는지 확인:
```dockerfile
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew
```

---

## 📋 체크리스트

### 외부망 작업 (캐시 생성)
- [ ] Java/JDK 설치 확인 (`java -version`)
- [ ] `./gradlew build --refresh-dependencies` 성공
- [ ] `~/.gradle/caches` 폴더 존재 확인
- [ ] `gradle-cache.tar.gz` 생성 완료 (~500MB 이상)
- [ ] `docker-images/` 폴더에 파일 복사

### 폐쇄망 작업 (캐시 사용)
- [ ] `gradle-cache.tar.gz` 파일 복사 완료
- [ ] Docker 이미지 로드 완료
- [ ] `docker compose up --build` 성공
- [ ] 서비스 정상 동작 확인

---

## 📁 관련 파일 구조

```
project-root/
├── docker-images/
│   ├── gradle-cache.tar.gz      # Gradle 캐시 (~576MB)
│   ├── gradle-cache-guide.md    # 이 문서
│   └── *.tar                    # Docker 이미지들
├── user-service/
│   └── Dockerfile               # 캐시 로직 포함
├── product-service/
│   └── Dockerfile               # 캐시 로직 포함
├── order-service/
│   └── Dockerfile               # 캐시 로직 포함
├── api-gateway/
│   └── Dockerfile               # 캐시 로직 포함
└── docker-compose.yml
```
