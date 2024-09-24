# Docker-Image-Optimization
## 도커 이미지 크기를 줄이는 법

1. 더 작은 base image 사용하기
    
    ```
    TAG                                  SIZE
    debian                               124.0MB
    alpine                               5.0MB
    gcr.io/distroless/static-debian11    2.51MB
    ```
    
2. 명령 결합해서 layer 수 줄이기
    
    ```
    # 기존 방법
    FROM python:3.9 as build
    ...
    # 필요한 패키지를 다운로드
    RUN pip install -r requirements.txt
    # 패키지 검증 (필요시)
    RUN pip check
    
    ------------------------------------------------
    
    # 새로운 방법(결합하기)
    FROM python:3.9
    ...
    # 필요한 패키지를 다운로드 및 검증
    RUN pip install -r requirements.txt && pip check
    ```
    
    ```
    # 패키지 설치, 시스템 업데이트, 클린업을 한 레이어에서 수행
    FROM python:3.9-slim
    
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt && \
        apt-get update && \
        apt-get install -y --no-install-recommends some-package && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*
    ```
    
3. build, run 분리하기
    
    멀티 스테이지 빌드를 사용하여 빌드 의존성을 최종 이미지에서 제외
    
    ```
    # 빌드 스테이지
    FROM python:3.9 AS builder
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --user -r requirements.txt
    
    # 실행 스테이지
    FROM python:3.9-slim
    WORKDIR /app
    COPY --from=builder /root/.local /root/.local
    COPY . .
    ENV PATH=/root/.local/bin:$PATH
    CMD ["python", "app.py"]
    ```
    
4. .dockerignore 사용
    
    불필요한 파일을 이미지에 포함하지 않기 위해서 필요함
    
    .dockerignore 파일 예시:
    
    ```
    Copy
    **/.git
    **/.gitignore
    **/.vscode
    **/coverage
    **/.env
    **/.aws
    **/.ssh
    Dockerfile
    README.md
    docker-compose.yml
    **/*.pyc
    **/*.pyo
    **/*.pyd
    **/__pycache__
    **/.pytest_cache
    **/.mypy_cache
    ```
    
5. Docker Slim과 같은 이미지 압축 툴 사용하기
6. 경량 패키지 관리자 사용하기
    
    예를 들어, Ubuntu에서 apt-get 대신 apt 사용
    
7. *사용하지 않는 파일 및 캐시 제거*
    
    ```
    RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*
    ```
<br>
### 최적화 적용 예

<hr>
최적화 적용 전 Dockerfile

```bash
# 큰 base 이미지 사용
FROM openjdk:11

# 작업 디렉토리 설정
WORKDIR /app

# 소스 코드 복사
COPY . .

# Java 애플리케이션 컴파일
RUN javac ImageTest.java

# 애플리케이션 실행
CMD ["java", "ImageTest"]
```

<hr>

최적화 적용 후 Dockerfile

``` bash
# 멀티 스테이지 빌드 사용

# 빌드 스테이지
FROM openjdk:11-jdk-slim as builder

# 작업 디렉토리 설정
WORKDIR /app

# 소스 코드만 복사 (불필요한 파일 제외)
COPY ImageTest.java .

# Java 애플리케이션 컴파일
RUN javac ImageTest.java

# 실행 스테이지
FROM openjdk:11-jre-slim

# 작업 디렉토리 설정
WORKDIR /app

# 빌드 스테이지에서 컴파일된 클래스 파일만 복사
COPY --from=builder /app/ImageTest.class .

# 환경 변수 설정
ENV JAVA_OPTS="-Xms64m -Xmx128m"

# 사용하지 않는 파일 및 캐시 제거
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 비루트 사용자로 실행
RUN groupadd -r javauser && useradd --no-log-init -r -g javauser javauser
USER javauser

# 애플리케이션 실행
CMD ["sh", "-c", "java $JAVA_OPTS ImageTest"]
```
   
