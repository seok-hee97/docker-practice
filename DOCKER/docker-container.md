# Docker Container

> 컨테이너 라이프사이클, 실행 옵션, 관리 방법 정리  
> 공식 문서: https://docs.docker.com/engine/containers/

---

## 목차

1. [컨테이너 라이프사이클](#1-컨테이너-라이프사이클)
2. [docker run 옵션](#2-docker-run-옵션)
3. [컨테이너 상태 관리](#3-컨테이너-상태-관리)
4. [컨테이너 내부 접근](#4-컨테이너-내부-접근)
5. [리소스 제한](#5-리소스-제한)

---

## 1. 컨테이너 라이프사이클

```
docker run ──▶ Created ──▶ Running ──▶ Stopped ──▶ Removed
                              │                       ▲
                              └── Paused ─────────────┘
```

| 상태 | 설명 | 전환 명령어 |
|------|------|-----------|
| Created | 생성됨, 아직 실행 안 됨 | `docker create` |
| Running | 실행 중 | `docker start` / `docker run` |
| Paused | 일시 정지 (프로세스 유지) | `docker pause` |
| Stopped | 종료됨 (프로세스 없음) | `docker stop` / `docker kill` |
| Removed | 삭제됨 | `docker rm` |

> 공식 문서: https://docs.docker.com/engine/containers/run/

---

## 2. docker run 옵션

### 2.1 기본 실행

```bash
docker run [옵션] <이미지명> [명령어]

# 예시
docker run nginx                          # 포그라운드 실행
docker run -d nginx                       # 백그라운드 실행
docker run -d --name my-nginx nginx       # 컨테이너 이름 지정
docker run --rm nginx                     # 종료 시 자동 삭제
```

---

### 2.2 포트 / 볼륨 / 환경변수

```bash
# 포트 매핑
docker run -p 8080:80 nginx              # 호스트 8080 → 컨테이너 80
docker run -p 127.0.0.1:8080:80 nginx   # 루프백 인터페이스에만 바인딩

# 볼륨 마운트
docker run -v ./data:/app/data nginx     # Bind Mount
docker run -v myvolume:/app/data nginx   # Named Volume

# 환경변수
docker run -e NODE_ENV=production app
docker run --env-file .env app           # .env 파일에서 읽기
```

---

### 2.3 네트워크

```bash
# 기본 브리지 네트워크
docker run -d nginx

# 커스텀 네트워크 지정
docker run -d --network my-network nginx

# 호스트 네트워크 사용
docker run -d --network host nginx

# 네트워크 없음
docker run -d --network none nginx
```

---

### 2.4 주요 옵션 정리

| 옵션 | 설명 |
|------|------|
| `-d` | 백그라운드 실행 (detached) |
| `-it` | 인터랙티브 터미널 연결 |
| `--name` | 컨테이너 이름 지정 |
| `--rm` | 종료 시 자동 삭제 |
| `-p HOST:CONTAINER` | 포트 매핑 |
| `-v HOST:CONTAINER` | 볼륨 마운트 |
| `-e KEY=VALUE` | 환경변수 설정 |
| `--env-file` | .env 파일로 환경변수 설정 |
| `--network` | 네트워크 지정 |
| `--restart` | 재시작 정책 설정 |
| `--memory` | 메모리 제한 |
| `--cpus` | CPU 제한 |

> 공식 문서: https://docs.docker.com/reference/cli/docker/container/run/

---

## 3. 컨테이너 상태 관리

```bash
# 목록 확인
docker ps              # 실행 중인 컨테이너
docker ps -a           # 전체 컨테이너 (종료 포함)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"  # 포맷 지정

# 시작 / 정지
docker start <컨테이너명>    # 중지된 컨테이너 시작
docker stop <컨테이너명>     # 정상 종료 (SIGTERM → 10초 후 SIGKILL)
docker kill <컨테이너명>     # 강제 종료 (SIGKILL 즉시 전송)
docker restart <컨테이너명>  # 재시작

# 일시 정지
docker pause <컨테이너명>    # 프로세스 일시 정지 (메모리 유지)
docker unpause <컨테이너명>  # 재개

# 삭제
docker rm <컨테이너명>        # 중지된 컨테이너 삭제
docker rm -f <컨테이너명>     # 실행 중인 컨테이너 강제 삭제
docker container prune        # 중지된 컨테이너 전체 삭제
```

---

## 4. 컨테이너 내부 접근

### 4.1 exec — 실행 중인 컨테이너에 명령어 실행

```bash
# 쉘 접속
docker exec -it <컨테이너명> /bin/bash
docker exec -it <컨테이너명> /bin/sh    # bash가 없는 경우 (alpine 등)

# 단일 명령어 실행
docker exec <컨테이너명> ls /app
docker exec <컨테이너명> cat /etc/hosts

# 환경변수 확인
docker exec <컨테이너명> env
```

---

### 4.2 logs — 로그 확인

```bash
docker logs <컨테이너명>                # 전체 로그 출력
docker logs -f <컨테이너명>             # 실시간 로그 스트리밍
docker logs --tail=100 <컨테이너명>     # 마지막 100줄
docker logs --since=1h <컨테이너명>     # 최근 1시간 로그
docker logs -t <컨테이너명>             # 타임스탬프 포함
```

---

### 4.3 inspect — 컨테이너 상세 정보

```bash
docker inspect <컨테이너명>                          # 전체 정보 (JSON)
docker inspect -f '{{.NetworkSettings.IPAddress}}' <컨테이너명>  # IP 주소만 출력
docker inspect -f '{{.State.Status}}' <컨테이너명>  # 상태만 출력
```

---

### 4.4 cp — 파일 복사

```bash
# 호스트 → 컨테이너
docker cp ./file.txt <컨테이너명>:/app/file.txt

# 컨테이너 → 호스트
docker cp <컨테이너명>:/app/logs/error.log ./error.log
```

---

## 5. 리소스 제한

컨테이너가 호스트의 리소스를 과도하게 사용하지 않도록 제한할 수 있다.

```bash
# 메모리 제한
docker run --memory=512m nginx                  # 최대 512MB
docker run --memory=1g --memory-swap=1g nginx   # 스왑 포함 제한

# CPU 제한
docker run --cpus=0.5 nginx                     # CPU 코어 0.5개 할당
docker run --cpu-shares=512 nginx               # 상대적 가중치 (기본값: 1024)
```

**docker-compose에서 리소스 제한:**

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

> 공식 문서: https://docs.docker.com/engine/containers/resource_constraints/

---

## 참고 공식 문서

| 주제 | 링크 |
|------|------|
| 컨테이너 실행 | https://docs.docker.com/engine/containers/run/ |
| docker run 레퍼런스 | https://docs.docker.com/reference/cli/docker/container/run/ |
| 리소스 제한 | https://docs.docker.com/engine/containers/resource_constraints/ |
| docker exec | https://docs.docker.com/reference/cli/docker/container/exec/ |
| docker logs | https://docs.docker.com/reference/cli/docker/container/logs/ |
