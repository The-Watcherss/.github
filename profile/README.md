# 👁 Watchers

**CCTV 기반 산업 현장 안전 모니터링 시스템**

지게차와 작업자를 실시간으로 검출해서 안전모 미착용, 지게차 접근 등 위험 상황을 감지하는
YOLO 기반 컴퓨터 비전 시스템을 만들고 있습니다.

---

## 🔧 What we're building

자동 라벨링부터 모델 학습, 실시간 검출, 대시보드까지 전체 파이프라인을 직접 구축합니다.

- **자동 라벨링 파이프라인** — 수집된 영상에서 프레임을 추출하고 자동으로 라벨링
- **YOLO 학습 자동화** — 라벨링된 데이터로 지게차 / 작업자 / 안전모 착용·미착용 검출 모델 학습
- **채팅 기반 관리 도구** — MCP를 연동해 라벨링 현황, 학습 이력, 모델 버전을 자연어로 조회
- **실시간 대시보드** — Electron + React 프론트엔드에서 학습/검출 현황 모니터링

## 🏗 시스템 구조

```
Frontend (Electron + React)
        │
        ▼
   backend (FastAPI + PostgreSQL)  ──▶  mcp-backend (채팅 도구 서버)
        │
        ▼
  gpu-backend (YOLO 학습 + 자동 라벨링, GPU 서버 1대)
```

GPU 1대를 `device=0`(라벨링) / `device=1`(학습)로 나눠서 두 작업을 동시에 처리합니다.
각 컴포넌트는 독립적으로 배포되고 REST API로만 통신합니다.

## 📋 핵심 기능

- **자동 라벨링** — 영상 → 프레임 추출 → 자동 라벨링, 진행상황(KPI)을 실시간으로 조회
- **YOLO 학습 자동화** — 1차는 pretrained 모델, 2차부터는 직전 학습의 `best.pt`를 자동으로 이어받아 학습
- **버전별 모델 관리** — 학습 완료마다 `best_v1.pt`, `best_v2.pt`처럼 버전이 자동 부여되고, 대시보드에서 원하는 버전 선택 후 다운로드
- **채팅 기반 운영** — MCP 연동으로 "라벨링 현황 어때?", "학습 새로 돌려줘" 같은 자연어 요청 처리
- **프로젝트별 격리** — 데이터셋 경로, 검출 클래스, 학습 결과 저장 폴더가 프로젝트 단위로 완전히 분리

## 📦 주요 레포지토리

| 레포                                                    | 설명                                               |
| ------------------------------------------------------- | -------------------------------------------------- |
| [`watchers`](https://github.com/watchers-org/watchers) | 백엔드 · GPU 서버 · MCP 서버 · Docker 구성 전체 |

레포 안 주요 문서:

- `README.md` — 전체 구조, 실행 순서, 프로젝트별 격리 방식, 채팅-API 매핑
- `DOCKER.md` — 팀 서버 Docker 공유 환경 세팅
- `FRONTEND_API.md` — 프론트 연동용 전체 API 스펙

## 🛠 기술 스택

`FastAPI` · `PostgreSQL` · `SQLAlchemy` · `Alembic` · `YOLO (Ultralytics)` ·
`Model Context Protocol (MCP)` · `Electron` · `React` · `Docker`

## 👥 팀원

| 이름   | 역할 |
| ------ | ---- |
| 황영중 |      |
| 홍현경 |      |
| 김민혁 |      |
| 김다희 |      |

---

<details>
<summary><b>📖 전체 기술 문서 펼치기 (백엔드 / GPU 서버 / MCP 서버 상세)</b></summary>

# Watchers 백엔드 / GPU 서버 / MCP 서버

> 각 폴더에 그 폴더 전용 README가 따로 있음. 실행 방법/환경변수는 거기서 자세히 다룸:
>
> - `backend/README.md`
> - `gpu-backend/README.md`
> - `mcp-backend/README.md`
>
> **팀 전체가 공유하는 Docker 환경 구성: `DOCKER.md` (팀 서버 세팅은 여기부터 볼 것)**

## 폴더 구조

```
watchers/
├── docker-compose.yml    ← 팀 서버에서 db+backend+mcp+pgadmin 한 번에 띄우는 파일
├── DOCKER.md
├── FRONTEND_API.md
│
├── backend/        ← 백엔드 서버 컴퓨터에 배포
│   ├── main.py           FastAPI 진입점 + /chat 엔드포인트 (Claude+MCP 연동)
│   ├── database.py       PostgreSQL 연결 설정
│   ├── models.py         SQLAlchemy 모델 (classes, projects, project_classes, ...)
│   ├── schemas.py        Pydantic 요청/응답 스키마
│   ├── seed_classes.py   클래스 마스터 초기 데이터 (Alembic 초기 마이그레이션에도 포함됨)
│   ├── schema.sql         DB 스키마 원본 SQL (참고용)
│   ├── alembic.ini, alembic/   DB 마이그레이션
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── .env.example
│   └── routers/
│       ├── classes.py     GET/POST /classes
│       ├── projects.py    프로젝트 CRUD + 클래스 선택
│       ├── autolabel.py   라벨링 KPI, 영상 등록, 라벨링 시작 트리거
│       └── training.py    학습 5개 엔드포인트 + 버전 관리 + progress/report 콜백
│
├── gpu-backend/          ← GPU 서버(딥러닝 서버, 1대) 컴퓨터에 배포
│   ├── main.py            학습(/train) + 자동라벨링(/autolabel/run) 통합 API (포트 9000 하나)
│   │                        device 분리로 학습/라벨링 동시 실행 가능 (train_lock / label_lock)
│   ├── train_wrapper.py   기존 yolo CLI 명령 실행 + 진행상황/결과 백엔드 보고
│   ├── generate_data_yaml.py  프로젝트 선택 클래스로 data.yaml 자동 생성
│   ├── requirements.txt
│   └── .env.example
│
├── mcp-backend/           ← 독립 프로세스로 어디든 배포 가능 (백엔드와 통신만 되면 됨)
    ├── mcp_server.py      백엔드 API를 도구로 감싸는 MCP 서버 (포트 8100)
    ├── Dockerfile
    ├── requirements.txt
    └── .env.example

```

## 실행 순서

Docker로 팀 전체가 공유하는 방식(추천)은 `DOCKER.md` 참고. 아래는 로컬에서 하나씩 직접 띄울 때:

### 1. PostgreSQL 준비

```bash
createdb watchers_db
```

### 2. 백엔드

```bash
cd backend
cp .env.example .env       # 값 채우기
pip install -r requirements.txt --break-system-packages
alembic upgrade head        # 스키마 생성 + 클래스 마스터 초기 데이터 등록
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

### 3. GPU 서버 (딥러닝 서버 컴퓨터, 1대에서 학습+라벨링 같이 처리)

```bash
cd gpu-backend
cp .env.example .env
pip install -r requirements.txt --break-system-packages
uvicorn main:app --host 0.0.0.0 --port 9000
```

### 4. MCP 서버 (독립 프로세스)

```bash
cd mcp-backend
cp .env.example .env
pip install -r requirements.txt --break-system-packages
python mcp_server.py
```

## 흐름 요약

1. 사용자가 프론트 채팅창에 질문 → 백엔드 `/chat` 호출
2. 백엔드가 Claude API 호출하면서 MCP 서버(`mcp-backend`)를 도구로 등록
3. Claude가 필요시 MCP tool 실행 → MCP 서버는 백엔드 REST API만 호출 (DB 직접 접근 안 함)
4. GPU 서버는 학습 진행 중/완료 시 백엔드에 HTTP로 보고만 함 (`/training/{id}/progress`, `/training/report`)
5. 라벨링도 동일하게 진행상황을 백엔드에 보고 (`/autolabel/videos/{id}/progress`)

각 컴포넌트는 옆에 있는 것 하나(대개 백엔드)와만 통신하도록 설계되어 있어,
서버가 물리적으로 몇 대로 나뉘어 있어도 IP/포트만 맞추면 그대로 동작함.

## 알아둘 것: 프로젝트마다 다른 것 vs 공통인 것

| 프로젝트마다 다름                                        | 모든 프로젝트 공통                                                             |
| -------------------------------------------------------- | ------------------------------------------------------------------------------ |
| 데이터셋 경로 (`Project.dataset_path`)                 | GPU 서버 주소, device 번호                                                     |
| 검출 클래스 종류/개수/순서 (`project_classes`)         | MCP tool 목록                                                                  |
| data.yaml 내용 (프로젝트별로 매번 생성)                  | Docker 이미지, DB 스키마                                                       |
| 학습 하이퍼파라미터 (요청마다 다르게 지정 가능)          | 1차 학습 기본 pretrained 모델(`FIRST_TRAIN_BASE_MODEL`, 기본 `yolo26m.pt`) |
| base_weights (1차/2차 자동 판단, 프로젝트별로 독립 추적) |                                                                                |
| 학습 결과 저장 폴더                                      |                                                                                |

**학습 결과 저장 폴더도 프로젝트별로 분리됨**:

- GPU 서버: `RUNS_PROJECT_DIR/project_{project_id}/{run_folder}/`
- 백엔드 최종 보관: `STORAGE_ROOT/project_{project_id}/{run_folder}/`

여러 프로젝트를 동시에 운영해도 파일이 안 섞이고, 폴더만 봐도 어느 프로젝트 결과인지 바로 구분됨.

## 알아둘 것: GPU 1대를 학습/라벨링으로 나눠 쓰는 방식

GPU가 물리적으로 1대여도 `device=0`(라벨링), `device=1`(학습)으로 나눠서 **동시 실행**이 가능해.
그래서 `gpu-server/main.py`는 락을 하나가 아니라 `train_lock` / `label_lock` 두 개로 분리해뒀어.

- 학습끼리는 겹치면 안 됨 (`train_lock`)
- 라벨링끼리도 겹치면 안 됨 (`label_lock`)
- 학습과 라벨링은 서로 다른 GPU device를 쓰니까 동시에 가능

`.env`의 `TRAIN_DEVICE`, `LABEL_DEVICE`로 몇 번 GPU를 쓸지 지정.

## 알아둘 것: base_weights(시작 가중치) 자동 판단

`POST /training/run`에서 `base_weights`를 안 넘기면 백엔드가 자동으로 정함:

- 이 프로젝트에 **완료된 학습이 없으면** → `None`으로 GPU서버에 전달 → GPU서버가 `FIRST_TRAIN_BASE_MODEL`(기본 `yolo26m.pt`) 사용 (1차 학습)
- **완료된 학습이 있으면** → 가장 최근 완료 학습의 `best.pt` 경로를 자동으로 이어받음 (2차 학습부터)

## 알아둘 것: data.yaml 실제 포맷

`test` 없이 `train`/`val`만 쓰고, `names`는 index-name 매핑(dict)으로 생성됨:

```yaml
path: /home/bax/workspace/watchers/dataset/watchers_merged_dataset
train: images/train
val: images/val
nc: 4
names:
  0: forklift
  1: person
  2: helmet
  3: no_helmet
```

`path`는 프로젝트의 `dataset_path`를 그대로 사용. hold-out 정확도 검증은 별도 test 셋이 없어서
기본적으로 `val` split 기준으로 함(`HOLDOUT_SPLIT` 환경변수로 변경 가능). 나중에 test 셋을 따로
마련하면 `HOLDOUT_SPLIT=test`로 바꾸는 걸 추천.

## 버전별 모델 다운로드 (best_v1.pt, best_v2.pt ...)

학습이 완료될 때마다(`POST /training/report` 콜백 시점) 프로젝트 내 순번으로 `version`이 자동 부여됨
(1차 완료 → version=1, 2차 완료 → version=2 ...). 관련 엔드포인트:

| 엔드포인트                                         | 설명                                                     |
| -------------------------------------------------- | -------------------------------------------------------- |
| `GET /projects/{project_id}/models`              | 이 프로젝트의 다운로드 가능한 버전 목록 (완료된 것만)    |
| `GET /projects/{project_id}/models/{version}/pt` | 버전 번호로 바로 다운로드 (`best_v2.pt`)               |
| `GET /models/{run_id}/pt`                        | run_id로 다운로드 (파일명은 버전 있으면`best_v{n}.pt`) |

채팅에서는 MCP의 `list_model_versions(project_id)` / `get_model_download_info(project_id, version)` tool로 동일하게 처리됨.

## 기획안 5개 엔드포인트 ↔ 채팅 트리거 매핑

| 사용자가 채팅에 이렇게 물으면                           | MCP tool                                             | 백엔드 엔드포인트                                          |
| ------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------- |
| "라벨링 현황 어때?"                                     | `get_labeling_kpi`                                 | `GET /labeling/kpi`                                      |
| "학습 이력 보여줘"                                      | `get_training_history`                             | `GET /training/history`                                  |
| "N번 학습 상세히 보여줘"                                | `get_training_detail`                              | `GET /training/{id}`                                     |
| "학습 새로 돌려줘"                                      | `start_training`                                   | `POST /training/run`                                     |
| "다운받을 수 있는 모델 뭐 있어?" / "N차 버전 받고 싶어" | `list_model_versions`, `get_model_download_info` | `GET /projects/{id}/models`, `GET /models/{run_id}/pt` |

5개 다 `mcp-backend/mcp_server.py`에 tool로 등록돼 있어서, 프론트 채팅창에서 자연어로 물어보면
Claude가 알아서 맞는 tool을 호출해. 마지막 항목은 MCP가 파일 자체를 전달할 순 없어서
다운로드 가능한 URL만 안내하고, 실제 다운로드는 그 링크를 클릭(또는 대시보드 버튼)해서 받는 구조야.

## 주의: IP/경로는 실제 환경에 맞게 .env에서 반드시 교체할 것

- `GPU_SERVER_URL`, `MCP_SERVER_URL`, `BACKEND_URL`
- `RUNS_PROJECT_DIR` (GPU 서버의 실제 로컬 경로)

</details>

---
