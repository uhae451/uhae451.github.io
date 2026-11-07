---
title: "Docker로 PostgreSQL과 Spring Boot 실행하기 — Volume, Network, Dockerfile, Compose"
date: 2026-11-07 12:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, docker-compose, dockerfile, postgresql, spring-boot]
description: "Docker에서 PostgreSQL과 Spring Boot를 실행하고 Volume, Network, Dockerfile, Docker Compose로 구성하고 관리하는 실전 사용법을 정리한다."
toc: true
---

Docker를 사용해 PostgreSQL을 실행하고, Spring Boot 애플리케이션과 연결하는 과정 정리.

개별 컨테이너 실행부터 Volume, Network, Dockerfile을 거쳐 최종적으로 Docker Compose로 관리하는 것까지 다룬다.

## 1. 컨테이너 실행

Docker는 **이미지(Image)**를 기반으로 **컨테이너(Container)**를 생성한다.

- **Image:** 실행에 필요한 환경이 정의된 파일 묶음
- **Container:** 이미지를 기반으로 실행되는 독립적인 환경

### PostgreSQL 실행

```bash
docker run -d \
  --name pg-bookstore \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=db_project \
  -p 5433:5432 \
  postgres:16
```

여기서 기억할 옵션은 네 가지.

- `-d`: 백그라운드 실행
- `--name`: 컨테이너 이름 지정
- `-e`: 환경변수 전달
- `-p`: 호스트 포트와 컨테이너 포트 연결

`-p 5433:5432`는 로컬 PC의 5433 포트를 컨테이너의 5432 포트로 연결한다는 의미다.

따라서 로컬 Spring Boot에서는 다음 주소로 접근한다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5433/db_project
```

### 자주 사용하는 명령어

```bash
docker ps                  # 실행 중인 컨테이너
docker ps -a               # 전체 컨테이너
docker stop pg-bookstore   # 중지
docker start pg-bookstore  # 재실행
docker logs pg-bookstore   # 로그 확인
docker rm pg-bookstore     # 중지된 컨테이너 삭제
```

컨테이너 내부에 직접 접근할 수도 있다.

```bash
# 컨테이너 내부 터미널 접속
docker exec -it pg-bookstore bash

# PostgreSQL 직접 접속
docker exec -it pg-bookstore psql -U postgres -d db_project
```

`run`은 컨테이너를 새로 생성하고, `start`는 기존 컨테이너를 다시 실행한다.

## 2. Volume — 데이터 유지하기

DB 컨테이너를 삭제하면 컨테이너 내부에 저장한 데이터도 함께 사라질 수 있다.

이를 방지하기 위해 **Volume으로 데이터 저장 공간을 분리**한다.

### Volume 생성 및 연결

먼저 기존 컨테이너를 제거한다.

```bash
docker rm -f pg-bookstore
```

Volume 생성:

```bash
docker volume create pg-data
```

PostgreSQL 실행 시 Volume을 연결한다.

```bash
docker run -d \
  --name pg-bookstore \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=db_project \
  -p 5433:5432 \
  -v pg-data:/var/lib/postgresql/data \
  postgres:16
```

핵심은 이 부분이다.

```bash
-v pg-data:/var/lib/postgresql/data
```

- `pg-data`: Docker에서 관리하는 Volume
- `/var/lib/postgresql/data`: PostgreSQL 16의 데이터 저장 경로

이제 컨테이너를 삭제하더라도 `pg-data` Volume은 남아 있다.

동일한 Volume을 연결하여 컨테이너를 다시 생성하면 기존 DB 데이터를 사용할 수 있다.

### Volume 관리

```bash
docker volume ls              # Volume 목록
docker volume inspect pg-data # 상세 정보
docker volume rm pg-data      # Volume 삭제
```

**기억할 점**

- `docker rm`: 컨테이너 삭제. Named Volume은 유지된다.
- `docker volume rm`: 해당 Volume과 저장 데이터 삭제.

DB를 다시 생성했는데 기존 데이터가 그대로 보인다면, 연결된 Volume을 먼저 확인하면 된다.

## 3. Network — 컨테이너 연결하기

Spring Boot와 PostgreSQL을 각각 컨테이너로 실행한다면 두 컨테이너가 서로 통신할 수 있어야 한다.

이때 Docker Network를 사용한다.

### Network 생성

```bash
docker network create backend-net
```

기존 PostgreSQL 컨테이너를 네트워크에 연결한다.

```bash
docker network connect backend-net pg-bookstore
```

이제 같은 네트워크에 있는 다른 컨테이너에서 `pg-bookstore`라는 이름으로 DB에 접근할 수 있다.

### 컨테이너 간 통신

Spring Boot를 Docker 컨테이너로 실행하는 경우 DB 주소는 다음과 같다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://pg-bookstore:5432/db_project
```

로컬 실행과의 차이를 기억해야 한다.

- **로컬 PC에서 실행:** `localhost:5433`
- **같은 Docker Network에서 실행:** `pg-bookstore:5432`

컨테이너 내부에서 `localhost`는 호스트 PC나 다른 컨테이너가 아니라 자기 자신을 의미한다.

따라서 다른 컨테이너의 DB에 접근하려면 해당 컨테이너 이름과 내부 포트를 사용한다.

## 4. Dockerfile — Spring Boot 이미지 만들기

PostgreSQL은 공식 이미지를 사용했지만, 직접 작성한 Spring Boot 애플리케이션은 이미지를 생성해야 한다.

이를 정의하는 파일이 `Dockerfile`이다.

### Dockerfile 작성

Java 21 기반 Spring Boot 프로젝트 기준.

빌드 결과물의 이름을 `app.jar`로 고정한다.

```groovy
// build.gradle
tasks.named('bootJar') {
    archiveFileName = 'app.jar'
}
```

프로젝트 최상위에 `Dockerfile` 생성.

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY build/libs/app.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

각 명령어의 역할은 간단하다.

- `FROM`: 기반 이미지 지정
- `WORKDIR`: 작업 디렉터리 지정
- `COPY`: 파일 복사
- `ENTRYPOINT`: 컨테이너 실행 시 수행할 명령

### 이미지 빌드 및 실행

```bash
# Spring Boot JAR 생성
./gradlew bootJar

# Docker 이미지 생성
docker build -t bookstore-api:1.0 .
```

생성한 이미지로 컨테이너를 실행한다.

```bash
docker run -d \
  --name bookstore-api \
  --network backend-net \
  -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://pg-bookstore:5432/db_project \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=postgres \
  bookstore-api:1.0
```

`--network backend-net`을 통해 PostgreSQL과 동일한 네트워크에 연결한다.

이렇게 하면 Spring Boot와 PostgreSQL을 각각 독립된 컨테이너로 실행하면서도 서로 통신할 수 있다.

## 5. Docker Compose — 한 번에 실행하기

지금까지는 컨테이너마다 `docker run` 명령어를 작성했다.

컨테이너가 여러 개라면 포트, 환경변수, Volume, Network 등의 설정을 매번 입력해야 한다.

**Docker Compose는 이러한 실행 설정을 하나의 YAML 파일로 관리한다.**

### compose.yaml

기존에 실행한 `bookstore-api`, `pg-bookstore` 컨테이너는 포트 충돌을 피하기 위해 먼저 제거한다.

```bash
docker rm -f bookstore-api pg-bookstore
```

프로젝트 최상위에 `compose.yaml`을 작성한다.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: db_project
    ports:
      - "5433:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres -d db_project" ]
      interval: 5s
      retries: 10

  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/db_project
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend

volumes:
  pg-data:

networks:
  backend:
```

여기서 중요한 설정만 짚어보면 다음과 같다.

**`services`**

실행할 컨테이너 서비스를 정의한다. 여기서는 `db`, `app` 두 개의 서비스가 존재한다.

**`volumes`**

PostgreSQL 데이터를 영속적으로 저장한다. Compose에서 선언한 `pg-data`는 프로젝트별 Named Volume으로 관리되므로, 앞서 `docker volume create`로 생성했던 Volume과는 별개일 수 있다.

**`networks`**

두 서비스를 같은 네트워크에 연결한다. 이때 Spring Boot는 서비스 이름인 `db`를 호스트명으로 사용한다.

**`depends_on`**

`app`이 `db`에 의존한다는 설정이다. `service_healthy` 조건을 사용하면 DB의 Healthcheck가 성공한 후 애플리케이션을 시작한다.

### Compose 실행 및 관리

```bash
# Spring Boot 빌드
./gradlew bootJar

# 이미지 빌드 + 컨테이너 실행
docker compose up -d --build

# 실행 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f

# 특정 서비스 로그
docker compose logs -f app

# 서비스 중지
docker compose stop

# 컨테이너 및 네트워크 제거
docker compose down
```

**데이터 삭제에 주의해야 하는 명령어**

```bash
docker compose down -v
```

`down`은 기본적으로 Named Volume을 유지하지만, `-v`를 추가하면 Compose가 관리하는 Named Volume도 삭제한다.

PostgreSQL 데이터까지 삭제될 수 있으므로 구분해서 사용한다.

## 6. 마지막으로 기억할 것

Docker를 사용할 때 핵심적으로 확인할 부분은 다음과 같다.

- **Container:** 이미지를 기반으로 실행하는 독립적인 환경
- **Volume:** 컨테이너를 삭제해도 데이터를 유지하기 위한 저장소
- **Network:** 서로 다른 컨테이너를 연결하는 네트워크
- **Dockerfile:** 애플리케이션 이미지를 만드는 방법 정의
- **Docker Compose:** 여러 컨테이너의 실행 설정을 하나의 파일로 관리

특히 Spring Boot와 DB를 Docker로 실행할 때는 **DB 연결 주소, Volume 연결 여부, Compose 실행 설정**을 먼저 확인하면 된다.

## 참고 문서

- [Docker Compose 빌드 설정](https://docs.docker.com/reference/compose-file/build/)
