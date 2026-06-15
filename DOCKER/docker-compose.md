# Docker Compose

> 여러 컨테이너를 하나의 YAML 파일로 정의하고 함께 실행하는 도구  
> 공식 문서: https://docs.docker.com/compose/

---

## 목차

1. [docker-compose.yml 구조](#1-docker-composeyml-구조)
2. [서비스 옵션](#2-서비스-옵션)
3. [예시 — 단일 서비스](#3-예시--단일-서비스)
4. [예시 — 멀티 서비스 (Web + DB)](#4-예시--멀티-서비스-web--db)
5. [예시 — 개발 vs 운영 환경 분리](#5-예시--개발-vs-운영-환경-분리)
6. [주요 명령어](#6-주요-명령어)

---

## 1. docker-compose.yml 구조

```yaml
services:         # 실행할 컨테이너 서비스 정의
  서비스명:
    image: ...    # 사용할 이미지
    build: ...    # 또는 Dockerfile로 직접 빌드
    ports: ...
    volumes: ...
    environment: ...
    depends_on: ...

volumes:          # Named Volume 선언
  볼륨명:

networks:         # 커스텀 네트워크 선언 (선택)
  네트워크명:
```

---

## 2. 서비스 옵션

### 2.1 image vs build

```yaml
# 방법 1: Docker Hub 이미지 사용
services:
  app:
    image: nginx:latest
```

```yaml
# 방법 2: Dockerfile로 빌드 (현재 디렉토리)
services:
  app:
    build: .
```

```yaml
# 방법 3: 빌드 경로와 파일 명시
services:
  app:
    build:
      context: ./app            # 빌드 컨텍스트 (COPY의 기준 경로)
      dockerfile: Dockerfile.dev
```

---

### 2.2 volumes

```yaml
services:
  app:
    volumes:
      - ./src:/app/src          # Bind Mount: 호스트 경로 → 컨테이너 경로
      - db_data:/var/lib/data   # Named Volume
      - /app/node_modules       # 익명 볼륨 (컨테이너 내부만 사용)

volumes:
  db_data:                      # Named Volume은 최상위에 선언 필요
```

---

### 2.3 environment / env_file

```yaml
services:
  app:
    # 방법 1: 직접 선언
    environment:
      - NODE_ENV=production
      - PORT=3000

    # 방법 2: .env 파일 참조 (권장 — 크레덴셜 분리)
    env_file:
      - .env
```

```bash
# .env 파일 예시
NODE_ENV=production
PORT=3000
DB_HOST=db
DB_PASSWORD=secret
```

> `.env` 파일은 `.gitignore`에 추가해 저장소에 포함되지 않도록 한다.

---

### 2.4 depends_on + healthcheck

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy    # DB 준비 완료 후 app 시작

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s     # 체크 주기
      timeout: 5s       # 타임아웃
      retries: 5        # 재시도 횟수
      start_period: 30s # 초기 대기 시간
```

> `depends_on`만 사용하면 시작 순서만 보장되고 준비 완료는 보장되지 않는다.  
> 공식 문서: https://docs.docker.com/compose/how-tos/startup-order/

---

### 2.5 networks

별도의 네트워크를 정의해 서비스 간 통신을 격리할 수 있다.

```yaml
services:
  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend        # db는 backend 네트워크에만 속함 (외부 노출 차단)

networks:
  frontend:
  backend:
```

---

### 2.6 restart 정책

```yaml
services:
  app:
    restart: always          # 항상 재시작
    # restart: no            # 재시작 안 함 (기본값)
    # restart: on-failure    # 비정상 종료 시에만 재시작
    # restart: unless-stopped # 직접 중지하지 않는 한 재시작
```

---

## 3. 예시 — 단일 서비스

정적 파일을 서빙하는 Nginx 서버.

```yaml
# docker-compose.yml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro    # :ro = 읽기 전용 마운트
    restart: unless-stopped
```

```bash
docker compose up -d
# http://localhost:8080 접속
```

---

## 4. 예시 — 멀티 서비스 (Web + DB)

Node.js 앱 + PostgreSQL 구성.

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure

  db:
    image: postgres:15-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  db_data:
```

```dockerfile
# Dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json .
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 5. 예시 — 개발 vs 운영 환경 분리

`docker-compose.override.yml`을 사용해 환경별 설정을 분리한다.

```yaml
# docker-compose.yml (공통 설정)
services:
  app:
    build: .
    env_file:
      - .env

  db:
    image: postgres:15-alpine
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

```yaml
# docker-compose.override.yml (개발 환경 — 자동 적용)
services:
  app:
    volumes:
      - ./src:/app/src    # Bind Mount로 코드 실시간 반영
    environment:
      - NODE_ENV=development
    ports:
      - "3000:3000"

  db:
    ports:
      - "5432:5432"       # 개발 환경에서는 DB 포트 노출
```

```yaml
# docker-compose.prod.yml (운영 환경 — 명시적 지정 필요)
services:
  app:
    environment:
      - NODE_ENV=production
    restart: always
```

```bash
# 개발 환경 실행 (override 자동 적용)
docker compose up

# 운영 환경 실행 (파일 명시)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

> 공식 문서: https://docs.docker.com/compose/how-tos/multiple-compose-files/

---

## 6. 주요 명령어

```bash
# 서비스 시작
docker compose up -d               # 백그라운드 실행
docker compose up --build          # 이미지 재빌드 후 실행

# 서비스 종료
docker compose down                # 컨테이너 + 네트워크 제거
docker compose down -v             # 볼륨까지 제거 (데이터 초기화)

# 상태 확인
docker compose ps                  # 서비스 상태 목록
docker compose logs -f             # 실시간 로그
docker compose logs -f <서비스명>  # 특정 서비스 로그

# 개별 서비스 제어
docker compose build <서비스명>    # 특정 서비스만 재빌드
docker compose restart <서비스명>  # 특정 서비스 재시작
docker compose stop <서비스명>     # 특정 서비스 중지

# 컨테이너 내부 접속
docker compose exec <서비스명> /bin/sh

# 설정 검증
docker compose config              # 최종 병합된 설정 출력 (문법 오류 확인용)
```
