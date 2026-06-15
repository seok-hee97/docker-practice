# Docker Guide

> Docker 핵심 개념 및 사용 가이드  
> 공식 문서: https://docs.docker.com/

---

## 목차

1. [핵심 개념](#1-핵심-개념)
2. [Dockerfile](#2-dockerfile)
3. [Docker Compose](#3-docker-compose)
4. [주요 명령어](#4-주요-명령어)
5. [트러블슈팅](#5-트러블슈팅)
6. [참고 공식 문서](#6-참고-공식-문서)

---

## 1. 핵심 개념

### 1.1 Image vs Container

```
Dockerfile ──(build)──▶ Image ──(run)──▶ Container
```

| | Image | Container |
|---|---|---|
| 정의 | 실행 가능한 읽기 전용 템플릿 | Image를 실행한 인스턴스 |
| 상태 | 불변 (Immutable) | 실행 중 / 종료 |
| 비유 | 클래스 | 클래스의 인스턴스 |

> 공식 문서: https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/

---

### 1.2 컨테이너 네트워크

Docker는 컨테이너 간 통신을 위한 가상 네트워크를 제공한다.

**네트워크 드라이버 종류:**

| 드라이버 | 설명 | 기본값 |
|---------|------|:------:|
| `bridge` | 같은 호스트의 컨테이너 간 통신 | ✓ |
| `host` | 컨테이너가 호스트 네트워크를 직접 사용 | |
| `none` | 네트워크 비활성화 | |
| `overlay` | 여러 Docker 호스트 간 통신 (Swarm) | |

**Docker Compose 네트워크 규칙:**
- Compose는 기본적으로 하나의 `bridge` 네트워크를 자동 생성한다.
- 같은 Compose 내 서비스끼리는 **서비스 이름**을 호스트명으로 DNS 해석한다.
- 컨테이너 내부에서 `localhost`는 **자기 자신**만 가리킨다.

```yaml
services:
  app:
    environment:
      - DB_HOST=db    # "db" 는 아래 서비스 이름 → DNS 자동 해석
  db:
    image: postgres:15
```

> 공식 문서: https://docs.docker.com/engine/network/

---

### 1.3 Volume

컨테이너가 종료되면 내부 데이터는 사라진다. Volume을 사용하면 데이터를 영속적으로 유지할 수 있다.

**Volume 유형 비교:**

| 유형 | 선언 방식 | 특징 | 주요 용도 |
|------|----------|------|---------|
| Named Volume | `volume명:컨테이너경로` | Docker가 관리, 이식성 좋음 | DB 데이터 영속 저장 |
| Bind Mount | `호스트경로:컨테이너경로` | 호스트 파일 직접 연결 | 개발 중 코드 실시간 반영 |
| tmpfs Mount | 메모리에만 저장 | 재시작 시 삭제 | 민감 데이터 임시 저장 |

```yaml
volumes:
  - db_data:/var/lib/postgresql/data    # Named Volume
  - ./src:/app/src                      # Bind Mount
```

> 공식 문서: https://docs.docker.com/engine/storage/volumes/

---

### 1.4 환경변수 (Environment Variables)

설정값을 이미지나 코드에 하드코딩하지 않고 런타임에 주입하는 방법.

**방법 1 — `environment` 직접 선언:**
```yaml
services:
  app:
    environment:
      - DB_HOST=db
      - DB_PORT=5432
```

**방법 2 — `.env` 파일로 분리 (권장):**
```bash
# .env
DB_HOST=db
DB_PORT=5432
```
```yaml
services:
  app:
    env_file:
      - .env
```

> `.env` 파일은 반드시 `.gitignore`에 추가해 크레덴셜이 저장소에 포함되지 않도록 한다.

> 공식 문서: https://docs.docker.com/compose/how-tos/environment-variables/

---

### 1.5 Port Mapping

호스트의 포트와 컨테이너의 포트를 연결한다.

```yaml
ports:
  - "HOST_PORT:CONTAINER_PORT"

# 예시
ports:
  - "8080:80"    # 호스트 8080 → 컨테이너 80
  - "5432:5432"  # 호스트 5432 → 컨테이너 5432
```

> 외부에 노출할 필요 없는 서비스(내부 DB 등)는 `ports:` 를 생략해 불필요한 포트 노출을 막는다.

> 공식 문서: https://docs.docker.com/compose/how-tos/networking/

---

## 2. Dockerfile

### 2.1 이미지를 만드는 두 가지 방법

**방법 1 — 컨테이너를 직접 수정 후 커밋 (전통적 방식)**

1. 베이스 이미지(ubuntu, alpine 등)로 컨테이너 생성
2. 컨테이너 내부에서 패키지 설치 및 소스코드 복사
3. 동작을 확인하고 `docker commit`으로 이미지 생성

```bash
docker run -it ubuntu /bin/bash
# 내부에서 설정 작업 후
docker commit <컨테이너명> my-image:1.0
```

> 재현이 어렵고 이력 추적이 불가능해 현재는 Dockerfile 방식을 권장한다.

**방법 2 — Dockerfile (권장)**

빌드 과정을 코드로 기록하므로 재현 가능하고 버전 관리가 가능하다.

```bash
docker build -t my-image:1.0 .
```

---

### 2.2 기본 구조

```dockerfile
FROM node:20-slim            # 베이스 이미지

WORKDIR /app                 # 작업 디렉토리 설정

COPY package*.json .         # 의존성 파일 먼저 복사 (레이어 캐시 활용)
RUN npm ci

COPY . .                     # 소스 코드 복사

EXPOSE 3000                  # 컨테이너가 사용하는 포트 명시

CMD ["node", "server.js"]    # 컨테이너 시작 시 실행할 명령어
```

---

### 2.3 .dockerignore

빌드 컨텍스트에서 불필요한 파일을 제외해 이미지 크기를 줄이고 빌드 속도를 높인다.

```
# .dockerignore
.git
.env
node_modules
*.log
__pycache__
*.pyc
```

> `COPY . .` 실행 전에 적용되므로, 빌드 컨텍스트 자체가 작아진다.

---

### 2.4 레이어 캐시 최적화

Dockerfile의 각 명령어는 **레이어**를 생성한다.  
변경이 잦은 파일을 나중에 복사해야 캐시가 효율적으로 재사용된다.

```dockerfile
# 나쁜 예: 소스 변경 시 npm install도 매번 재실행됨
COPY . .
RUN npm install

# 좋은 예: package.json이 변경되지 않으면 npm install 캐시 재사용
COPY package*.json .
RUN npm install
COPY . .
```

---

### 2.5 멀티스테이지 빌드

빌드 환경과 실행 환경을 분리해 최종 이미지 크기를 줄이는 패턴.

```dockerfile
# 1단계: 빌드
FROM node:20 AS builder
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY . .
RUN npm run build

# 2단계: 실행 (빌드 결과물만 복사)
FROM node:20-slim AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json .
RUN npm ci --only=production
CMD ["node", "dist/server.js"]
```

> `builder` 스테이지의 개발 의존성과 소스코드는 최종 이미지에 포함되지 않는다.  
> 공식 문서: https://docs.docker.com/build/building/multi-stage/

---

### 2.6 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `FROM` | 베이스 이미지 지정 |
| `WORKDIR` | 작업 디렉토리 설정 (없으면 자동 생성) |
| `COPY` | 호스트 파일 → 이미지로 복사 |
| `ADD` | COPY + URL 다운로드 / tar 자동 해제 |
| `RUN` | 이미지 빌드 시 실행할 명령어 |
| `ENV` | 환경변수 설정 (빌드 & 런타임 모두 적용) |
| `ARG` | 빌드 시에만 사용되는 변수 |
| `EXPOSE` | 컨테이너가 사용하는 포트 문서화 (실제 오픈 아님) |
| `CMD` | 컨테이너 시작 시 실행할 기본 명령어 (override 가능) |
| `ENTRYPOINT` | 컨테이너 시작 시 항상 실행되는 명령어 |

> 공식 문서: https://docs.docker.com/reference/dockerfile/

---

## 3. Docker Compose

### 3.1 기본 구조

```yaml
# docker-compose.yml
services:
  web:
    build: .                              # Dockerfile로 이미지 빌드
    ports:
      - "8080:80"
    volumes:
      - ./src:/app/src                   # Bind Mount (개발용)
    environment:
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy       # DB 준비 완료 후 시작

  db:
    image: postgres:15
    volumes:
      - db_data:/var/lib/postgresql/data # Named Volume
    environment:
      - POSTGRES_PASSWORD=password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:                               # Named Volume 선언
```

---

### 3.2 depends_on과 healthcheck

`depends_on`은 컨테이너 **시작 순서**만 제어한다.  
DB 초기화 완료 여부까지 보장하려면 `healthcheck`와 함께 사용해야 한다.

| 방식 | 보장 범위 |
|------|---------|
| `depends_on: [db]` | db 컨테이너가 **시작**됨 |
| `depends_on: db: condition: service_healthy` | db가 **준비 완료**됨 |

```yaml
# MySQL healthcheck 예시
db:
  image: mysql:8
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 5

# PostgreSQL healthcheck 예시
db:
  image: postgres:15
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s
    timeout: 5s
    retries: 5
```

> 공식 문서: https://docs.docker.com/compose/how-tos/startup-order/

---

## 4. 주요 명령어

> **CLI V1 vs V2**  
> `docker-compose` (하이픈, 구버전 V1)는 deprecated 상태이며, 현재 표준은 `docker compose` (공백, V2 플러그인)이다.  
> 두 명령어의 문법은 동일하므로 아래 예시에서 `docker-compose` → `docker compose`로 바꿔 사용할 수 있다.

### 4.1 이미지 빌드 / 실행

```bash
# 이미지 빌드
docker compose build

# 컨테이너 시작 (백그라운드)
docker compose up -d

# 재빌드 후 시작
docker compose up --build

# 컨테이너 종료 및 네트워크 제거
docker compose down

# 컨테이너 종료 + 볼륨 삭제 (데이터 초기화)
docker compose down -v

# 특정 서비스만 재시작
docker compose restart <서비스명>
```

---

### 4.2 로그 확인

```bash
# 전체 서비스 로그
docker compose logs

# 실시간 로그 스트리밍
docker compose logs -f

# 특정 서비스 로그만
docker compose logs -f <서비스명>

# 마지막 N줄만 출력
docker compose logs --tail=100 <서비스명>

# 단일 컨테이너 로그
docker logs <컨테이너명>
docker logs -f <컨테이너명>
```

---

### 4.3 컨테이너 상태 확인 및 접속

```bash
# 실행 중인 컨테이너 목록
docker ps

# 전체 컨테이너 목록 (종료된 것 포함)
docker ps -a

# 컨테이너 내부 접속
docker exec -it <컨테이너명> /bin/bash
docker exec -it <컨테이너명> /bin/sh    # bash가 없는 경우

# 컨테이너 내부에서 명령어 실행
docker exec <컨테이너명> <명령어>
```

---

### 4.4 이미지 관리

```bash
# 이미지 목록 확인
docker images

# 특정 이미지 삭제
docker rmi <이미지명>

# 전체 이미지 삭제
docker images -q | xargs docker rmi

# 사용하지 않는 리소스 전체 정리 (컨테이너 / 이미지 / 네트워크 / 볼륨)
docker system prune -a
```

> 공식 문서: https://docs.docker.com/reference/cli/docker/

---

## 5. 트러블슈팅

### T1. 컨테이너 간 `localhost` 접근 불가

**증상:**
```
Can't connect to server on 'localhost' ([Errno 99] Cannot assign requested address)
```

**원인:**  
컨테이너 내부의 `localhost`는 컨테이너 자기 자신을 가리킨다.  
`DB_HOST=localhost`로 설정하면 다른 컨테이너(DB)가 아닌 자기 자신에 접근을 시도한다.

**해결:**  
`localhost` 대신 Docker Compose **서비스 이름** 사용

```yaml
# Before
environment:
  - DB_HOST=localhost

# After
environment:
  - DB_HOST=db    # docker-compose 서비스 이름
```

---

### T2. 볼륨 데이터 설정 충돌

**증상:**
```
[ERROR] Different settings for server and data directory.
[ERROR] Data Dictionary initialization failed.
```

**원인:**  
기존 볼륨 데이터가 남아있는 상태에서 DB 설정이 변경된 경우 충돌이 발생한다.

**해결:**  
볼륨 삭제 후 재기동 (운영 데이터가 있는 경우 백업 필수)

```bash
docker compose down -v    # 볼륨 포함 삭제
docker compose up
```

---

### T3. `depends_on`만으로 DB 연결 실패

**증상:**
```
Connection refused (컨테이너는 시작됐으나 DB가 아직 초기화 중)
```

**원인:**  
`depends_on`은 컨테이너 **시작**만 보장하며, DB 내부 초기화 완료는 보장하지 않는다.

**해결:**  
`healthcheck` + `condition: service_healthy` 조합 사용 → [3.2 참고](#32-depends_on과-healthcheck)

---

### T4. 코드 수정 후 변경사항 미반영

**증상:**  
코드를 수정했지만 컨테이너 재시작 후에도 이전 코드가 실행된다.

**원인:**  
`docker compose up`은 기존 이미지를 재사용한다.  
`COPY`로 이미지에 포함된 코드는 명시적으로 재빌드해야 갱신된다.

**해결:**
```bash
# 강제 재빌드 후 실행
docker compose up --build

# 특정 서비스만 재빌드
docker compose build <서비스명>
docker compose up -d <서비스명>
```

> 개발 환경에서는 Bind Mount(`- ./src:/app/src`)를 사용하면 재빌드 없이 코드 변경이 즉시 반영된다.

---

## 6. 참고 공식 문서

| 주제 | 링크 |
|------|------|
| Docker 전체 문서 | https://docs.docker.com/ |
| Docker 개념 (시작하기) | https://docs.docker.com/get-started/ |
| Dockerfile 레퍼런스 | https://docs.docker.com/reference/dockerfile/ |
| Docker Compose 레퍼런스 | https://docs.docker.com/compose/ |
| 컨테이너 네트워크 | https://docs.docker.com/engine/network/ |
| 볼륨 (Storage) | https://docs.docker.com/engine/storage/volumes/ |
| 환경변수 관리 | https://docs.docker.com/compose/how-tos/environment-variables/ |
| 서비스 시작 순서 제어 | https://docs.docker.com/compose/how-tos/startup-order/ |
| CLI 레퍼런스 | https://docs.docker.com/reference/cli/docker/ |
