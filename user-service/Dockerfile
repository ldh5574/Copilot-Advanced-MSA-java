FROM localhost:5000/temurin:17-jdk-alpine AS builder
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
COPY user-service user-service
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew

# 오프라인 모드로 빌드 (캐시가 있으면)
RUN if [ -d /root/.gradle/caches ]; then \
        ./gradlew :user-service:bootJar --no-daemon --offline || \
        ./gradlew :user-service:bootJar --no-daemon; \
    else \
        ./gradlew :user-service:bootJar --no-daemon; \
    fi

FROM localhost:5000/temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/user-service/build/libs/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
