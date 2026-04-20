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
COPY order-service order-service
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew

# 오프라인 모드로 빌드 (캐시가 있으면)
RUN if [ -d /root/.gradle/caches ]; then \
        ./gradlew :order-service:bootJar --no-daemon --offline || \
        ./gradlew :order-service:bootJar --no-daemon; \
    else \
        ./gradlew :order-service:bootJar --no-daemon; \
    fi

FROM localhost:5000/temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/order-service/build/libs/*.jar app.jar
EXPOSE 8083
ENTRYPOINT ["java", "-jar", "app.jar"]
