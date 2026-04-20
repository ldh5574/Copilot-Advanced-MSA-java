# Docker Images 다운로드 가이드

폐쇄망 환경에서 MSA 프로젝트를 실행하기 위한 Docker 이미지 및 Gradle 캐시 다운로드 경로입니다.

---

## 📦 다운로드 파일 목록

다음 파일들을 아래 링크에서 다운로드할 수 있습니다.

| 파일명 | 용도 | 공식 링크 |
|--------|------|---------|
| `gradle-cache.tar.gz` | Gradle 의존성 캐시 | [Maven Central](https://repo.maven.apache.org/maven2/) · [Gradle Plugin Portal](https://plugins.gradle.org/) |
| `zookeeper.tar` | Zookeeper 이미지 | [Confluent Zookeeper](https://hub.docker.com/r/confluentinc/cp-zookeeper) |
| `kafka.tar` | Kafka 이미지 | [Confluent Kafka](https://hub.docker.com/r/confluentinc/cp-kafka) |
| `kafka-ui.tar` | Kafka UI 이미지 | [Kafka UI](https://hub.docker.com/r/provectuslabs/kafka-ui) |
| `registry.tar` | Docker Registry 이미지 | [Docker Registry](https://hub.docker.com/_/registry) |
| `registry-data.tar.gz` | Docker Registry 데이터 | [Docker Registry](https://hub.docker.com/_/registry) |
| `temurin-jdk.tar` | Java 17 JDK | [Eclipse Temurin](https://hub.docker.com/_/eclipse-temurin) |
| `temurin-jre.tar` | Java 17 JRE | [Eclipse Temurin](https://hub.docker.com/_/eclipse-temurin) |

