# Watchers — Docker로 팀 공유 환경 구성하기

## 핵심 원칙

**PG만 Docker로 올리는 게 아니라, 백엔드 자체를 팀 전체가 공유하는 하나의 서버로 만드는 것**.
팀원 각자 로컬에서 백엔드를 따로 띄우면 프로젝트/학습이력이 사람마다 다르게 쌓여서
"내 컴퓨터엔 있는데 왜 다른 사람 화면엔 없지" 같은 문제가 생김.

구조:

```
[팀 서버 1대] (또는 클라우드 VM)
├── docker-compose로 다음 4개를 한 번에 띄움
│   ├── db       (PostgreSQL, 5432)
│   ├── backend  (FastAPI, 8000)
│   ├── mcp      (MCP 서버, 8100)
│   └── pgadmin  (DB 조회용 웹 UI, 5050)
│
[팀원 각자 컴퓨터]
└── 프론트(Electron+React)만 로컬에서 실행하고, .env의 REACT_APP_BACKEND_URL이
    팀 서버의 IP:8000을 가리키도록 설정

[GPU 서버] (물리적으로 별도, GPU 있는 컴퓨터)
└── 지금처럼 그대로 bare-metal로 실행 (Docker 안 씀), .env의 BACKEND_URL이 팀 서버 IP를 가리킴
```

## 실행 방법 (팀 서버에서 딱 1번)

```bash
cd watchers
cp .env.example .env
# .env 열어서 ANTHROPIC_API_KEY 채우기

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
- **DB 조회(pgAdmin)**: `http://<팀서버IP>:5050` — 로그인 후 서버 등록 시 호스트는 `db`(컨테이너 내부 이름), 포트 5432, 계정은 docker-compose.yml의 `POSTGRES_USER`/`PASSWORD`
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
GPU_SERVER_URL: http://<GPU서버IP>:9000
```

이 값을 실제 GPU 서버 IP로 바꿔야 함. GPU 서버 자체는 Docker로 안 옮기는 걸 추천 —
CUDA/드라이버 버전 맞추는 게 번거롭고, 어차피 물리적으로 그 컴퓨터에서만 학습이 도니까
지금처럼 `gpu-server/README.md`대로 venv + uvicorn으로 직접 실행하면 됨.

(굳이 GPU 서버도 컨테이너화하고 싶다면 `nvidia-container-toolkit` 설치하고
`nvidia/cuda` 베이스 이미지로 Dockerfile을 따로 만들 수 있음 — 필요하면 요청해줘.)

## 데이터 유지 (컨테이너 재시작해도 안 날아감)

`docker-compose.yml`에 3개의 named volume이 있음:

- `pgdata` — PostgreSQL 데이터
- `storage_runs` — 학습 완료된 `best.pt`, `results.json`
- `pgadmin_data` — pgAdmin 설정

`docker compose down`은 컨테이너만 지우고 volume은 남음. **`docker compose down -v`는 volume까지
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

Alembic 세팅해뒀음. `main.py`는 더 이상 `create_all()`로 자동 생성하지 않고,
`watchers-backend/Dockerfile`이 컨테이너 시작할 때 `alembic upgrade head`를 자동으로 실행해서
스키마를 맞춤. `docker compose up`만 하면 DB 스키마 + 클래스 마스터 초기 데이터까지 다 준비됨.

팀원 중 누군가 `models.py`를 수정하면:

```bash
cd watchers-backend
alembic revision --autogenerate -m "설명"
alembic upgrade head
```

로 마이그레이션 파일을 만들어서 git에 커밋. 다른 팀원은 `git pull` 받고
`docker compose up -d --build`만 다시 하면(Dockerfile이 자동으로 `alembic upgrade head` 실행)
스키마가 자동으로 맞춰짐 — 수동으로 `ALTER TABLE` 칠 필요 없음.

로컬에서 Docker 없이 직접 돌릴 때도 서버 켜기 전에 `alembic upgrade head` 한 번 실행하면 됨
(`watchers-backend/README.md` 참고).

## 정리: 공유해야 할 것 체크리스트

- [X] PostgreSQL (docker-compose의 `db`)
- [X] 백엔드 API (docker-compose의 `backend`) — 팀원 각자 로컬 백엔드 띄우지 말 것
- [X] MCP 서버 (docker-compose의 `mcp`)
- [X] 학습 결과 파일 저장소 (`storage_runs` volume)
- [X] `.env.example` 파일들을 git에 커밋 (실제 비밀값 `.env`는 커밋 금지, `.gitignore` 반영해둠)
- [X] Alembic 마이그레이션 (세팅 완료, `alembic/versions/0001_initial.py`)
- [X] GPU 서버는 공유 대상 아님 — 물리적으로 1대뿐이라 그대로 유지
