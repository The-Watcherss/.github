# Watchers — Docker로 팀 공유 환경 구성하기 / Shared Team Environment via Docker

[🇰🇷 한국어](#한국어) ・ [🇺🇸 English](#english)

---

<br>

# 한국어

## 핵심 원칙

**PostgreSQL만 Docker로 올리는 게 아니라, 백엔드 자체를 팀 전체가 공유하는 하나의 서버로 만드는 것**.
팀원 각자 로컬에서 백엔드를 따로 띄우면 프로젝트/학습이력이 사람마다 다르게 쌓여서
"내 컴퓨터엔 있는데 왜 다른 사람 화면엔 없지" 같은 문제가 생긴다.

구조:

```
[팀 서버 1대] (또는 클라우드 VM)
├── docker-compose로 다음 4개를 한 번에 띄움
│   ├── db       (PostgreSQL 16, 5432)
│   ├── backend  (FastAPI, 8000)
│   ├── mcp      (MCP 서버, 8100)
│   └── pgadmin  (DB 조회용 웹 UI, 5050)
│
[팀원 각자 컴퓨터]
└── 프론트(Electron+React)만 로컬에서 실행하고, .env의 VITE_API_BASE_URL이
    팀 서버의 IP:8000을 가리키도록 설정

[GPU 서버] (물리적으로 별도, GPU 있는 컴퓨터)
└── 지금처럼 그대로 bare-metal로 실행 (Docker 안 씀), .env의 BACKEND_URL이 팀 서버 IP를 가리킴
```

## 실행 방법 (팀 서버에서 딱 1번)

```bash
cd watchers
cp .env.example .env
# .env 열어서 채우기: DATABASE_URL, GPU_SERVER_URL, MCP_SERVER_URL,
# OPENAI_API_KEY, OPENAI_CHAT_MODEL, STORAGE_ROOT, BACKEND_URL

docker compose up -d --build
```

이후 확인:

```bash
docker compose ps
docker compose logs -f backend
```

## 팀원들이 접속하는 방법

- **백엔드 API**: `http://<팀서버IP>:8000` — 프론트/Postman 등에서 이 주소로 호출
- **API 문서(Swagger)**: `http://<팀서버IP>:8000/docs` — 브라우저로 열면 전체 엔드포인트 확인 가능
- **DB 조회(pgAdmin)**: `http://<팀서버IP>:5050` — 로그인 후 서버 등록 시 호스트는 `db`(컨테이너 내부 이름), 포트 5432, 계정은 `POSTGRES_USER`/`POSTGRES_PASSWORD`(기본 `watchers_user`)
- **DB 직접 접속(DBeaver 등)**: 호스트 `<팀서버IP>`, 포트 5432, 계정 동일

## 비밀번호는 꼭 바꿀 것

`docker-compose.yml` 안의 아래 값들은 예시값이니 실제 배포 전에 바꾸기:

```yaml
POSTGRES_PASSWORD: watchers_password_change_me
PGADMIN_DEFAULT_PASSWORD: pgadmin_password_change_me
```

## GPU 서버 연결

`docker-compose.yml`의 `backend` 서비스 환경변수 중:

```yaml
GPU_SERVER_URL: ${GPU_SERVER_URL} # 예: http://<GPU서버IP>:9000
```

이 값을 실제 GPU 서버 IP로 바꿔야 함(`.env`에서). GPU 서버 자체는 Docker로 안 옮기는 걸 추천 —
CUDA/드라이버 버전 맞추는 게 번거롭고, 어차피 물리적으로 그 컴퓨터에서만 학습이 도니까
`gpu-backend/README.md`대로 venv + uvicorn으로 직접 실행하면 된다.

## 데이터 유지 (컨테이너 재시작해도 안 날아감)

`docker-compose.yml`에 3개의 named volume이 있다:

- `pgdata` — PostgreSQL 데이터
- `storage_runs` — 학습 완료된 `best.pt`, `results.json` (backend 컨테이너의 `/app/storage/runs`에 마운트)
- `pgadmin_data` — pgAdmin 설정

`docker compose down`은 컨테이너만 지우고 volume은 남는다. **`docker compose down -v`는 volume까지
지우니까 절대 실수로 치면 안 됨** (DB 데이터 다 날아감).

## 백업 (팀 데이터라 꼭 필요)

```bash
# DB 백업
docker compose exec db pg_dump -U watchers_user watchers_db > backup_$(date +%Y%m%d).sql

# 복원
cat backup_20260709.sql | docker compose exec -T db psql -U watchers_user watchers_db
```

주기적으로(cron 등) 백업해두는 걸 추천, 특히 학습 이력이 쌓이기 시작하면.

## 스키마 변경 시 주의 (Alembic 세팅 완료)

Alembic 세팅되어 있음. `main.py`는 `create_all()`로 자동 생성하지 않고,
`backend/Dockerfile`이 컨테이너 시작할 때 `alembic upgrade head`를 자동으로 실행해서
스키마를 맞춘다. `docker compose up`만 하면 DB 스키마 + 클래스 마스터 초기 데이터까지 다 준비된다.

팀원 중 누군가 `models.py`를 수정하면:

```bash
cd backend
alembic revision --autogenerate -m "설명"
alembic upgrade head
```

로 마이그레이션 파일을 만들어서 git에 커밋. 다른 팀원은 `git pull` 받고
`docker compose up -d --build`만 다시 하면(Dockerfile이 자동으로 `alembic upgrade head` 실행)
스키마가 자동으로 맞춰진다 — 수동으로 `ALTER TABLE` 칠 필요 없음.

로컬에서 Docker 없이 직접 돌릴 때도 서버 켜기 전에 `alembic upgrade head` 한 번 실행하면 된다
(`backend/README.md` 참고).

## 정리: 공유해야 할 것 체크리스트

- [x] PostgreSQL (docker-compose의 `db`)
- [x] 백엔드 API (docker-compose의 `backend`) — 팀원 각자 로컬 백엔드 띄우지 말 것
- [x] MCP 서버 (docker-compose의 `mcp`)
- [x] 학습 결과 파일 저장소 (`storage_runs` volume)
- [x] `.env.example` 파일들을 git에 커밋 (실제 비밀값 `.env`는 커밋 금지, `.gitignore` 반영해둠)
- [x] Alembic 마이그레이션 (세팅 완료, `backend/alembic/versions/`)
- [x] GPU 서버는 공유 대상 아님 — 물리적으로 1대뿐이라 bare-metal 그대로 유지

---

<br>

# English

## Core Principle

**This isn't just running PostgreSQL in Docker — it makes the backend itself one shared server the whole team uses.**
If every team member runs their own local backend, projects and training history pile up differently
for each person, leading to "it's on my machine, why isn't it on theirs?" problems.

Structure:

```
[One team server] (or a cloud VM)
├── docker-compose brings up 4 containers at once
│   ├── db       (PostgreSQL 16, 5432)
│   ├── backend  (FastAPI, 8000)
│   ├── mcp      (MCP server, 8100)
│   └── pgadmin  (web UI for DB inspection, 5050)
│
[Each team member's machine]
└── Only the frontend (Electron+React) runs locally, with .env's VITE_API_BASE_URL
    pointing at the team server's IP:8000

[GPU server] (a physically separate, GPU-equipped machine)
└── Runs bare-metal as before (not containerized); its .env BACKEND_URL points at the team server
```

## How to Run (once, on the team server)

```bash
cd watchers
cp .env.example .env
# fill in .env: DATABASE_URL, GPU_SERVER_URL, MCP_SERVER_URL,
# OPENAI_API_KEY, OPENAI_CHAT_MODEL, STORAGE_ROOT, BACKEND_URL

docker compose up -d --build
```

Then verify:

```bash
docker compose ps
docker compose logs -f backend
```

## How Team Members Connect

- **Backend API**: `http://<team-server-IP>:8000` — call this address from the frontend/Postman/etc.
- **API docs (Swagger)**: `http://<team-server-IP>:8000/docs` — open in a browser to see every endpoint
- **DB inspection (pgAdmin)**: `http://<team-server-IP>:5050` — after logging in, register a server with host `db` (the container's internal name), port 5432, credentials from `POSTGRES_USER`/`POSTGRES_PASSWORD` (default `watchers_user`)
- **Direct DB access (DBeaver, etc.)**: host `<team-server-IP>`, port 5432, same credentials

## Change the Default Passwords

The following values in `docker-compose.yml` are placeholders — change them before real deployment:

```yaml
POSTGRES_PASSWORD: watchers_password_change_me
PGADMIN_DEFAULT_PASSWORD: pgadmin_password_change_me
```

## Connecting the GPU Server

In the `backend` service's environment variables in `docker-compose.yml`:

```yaml
GPU_SERVER_URL: ${GPU_SERVER_URL} # e.g. http://<GPU-server-IP>:9000
```

Set this (in `.env`) to the real GPU server's IP. It's recommended **not** to containerize the GPU server itself —
matching CUDA/driver versions inside a container is a hassle, and training only ever runs on that one physical
machine anyway, so running it directly with venv + uvicorn as described in `gpu-backend/README.md` is simpler.

## Data Persistence (survives container restarts)

`docker-compose.yml` defines 3 named volumes:

- `pgdata` — PostgreSQL data
- `storage_runs` — completed training artifacts (`best.pt`, `results.json`), mounted at `/app/storage/runs` in the backend container
- `pgadmin_data` — pgAdmin settings

`docker compose down` removes containers but keeps the volumes. **`docker compose down -v` also deletes the
volumes — never run this by mistake** (it wipes the database).

## Backups (essential — this is shared team data)

```bash
# Back up the DB
docker compose exec db pg_dump -U watchers_user watchers_db > backup_$(date +%Y%m%d).sql

# Restore
cat backup_20260709.sql | docker compose exec -T db psql -U watchers_user watchers_db
```

Recommend backing up on a schedule (cron, etc.), especially once training history starts accumulating.

## Schema Changes (Alembic is already set up)

Alembic is configured. `main.py` does **not** auto-create tables via `create_all()`; instead
`backend/Dockerfile` automatically runs `alembic upgrade head` when the container starts, so the
schema is always kept in sync. Running `docker compose up` alone provisions the full DB schema plus
the seeded class master data.

When someone edits `models.py`:

```bash
cd backend
alembic revision --autogenerate -m "description"
alembic upgrade head
```

Commit the generated migration file to git. Other team members just `git pull` and re-run
`docker compose up -d --build` (the Dockerfile automatically runs `alembic upgrade head`) —
the schema updates itself, no manual `ALTER TABLE` needed.

Running locally without Docker also just needs one `alembic upgrade head` before starting the server
(see `backend/README.md`).

## Checklist: What Needs to Be Shared

- [x] PostgreSQL (docker-compose's `db`)
- [x] Backend API (docker-compose's `backend`) — don't run individual local backends
- [x] MCP server (docker-compose's `mcp`)
- [x] Training artifact storage (`storage_runs` volume)
- [x] `.env.example` files committed to git (real secrets in `.env` must never be committed — already in `.gitignore`)
- [x] Alembic migrations (set up, in `backend/alembic/versions/`)
- [x] The GPU server is *not* shared infrastructure — it's a single physical machine, kept bare-metal
