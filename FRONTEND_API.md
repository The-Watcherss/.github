# Watchers Frontend 연동 API 문서

프론트(Electron + React) 팀이 백엔드와 연동할 때 참고하는 문서.
Base URL은 백엔드 서버 주소 (`.env`의 `REACT_APP_BACKEND_URL`로 관리 권장, 예: `http://<백엔드IP>:8000`)

모든 응답은 JSON. 에러는 `{"detail": "에러 메시지"}` 형태로 옴 (FastAPI 기본 포맷), 상태 코드는 4xx/5xx.

---

## 0. 헬스체크

```
GET /health
→ { "status": "ok" }
```

---

## 1. 클래스

### 클래스 목록 (프로젝트 생성 화면 체크박스용)

```
GET /classes
→ [ { "id": 1, "name": "forklift", "display_name": "지게차" }, ... ]
```

### 새 클래스 추가 (관리자용, 보통 프론트에서 직접 쓸 일은 적음)

```
POST /classes
body: { "name": "fire-extinguisher", "display_name": "소화기" }
→ { "id": 5, "name": "fire-extinguisher", "display_name": "소화기" }
```

---

## 2. 프로젝트

### 프로젝트 생성 (사용자가 클래스 체크박스로 선택)

```
POST /projects
body: {
  "name": "A공장 안전모니터링",
  "description": "지게차/작업자 검출",
  "dataset_path": "/home/bax/workspace/1_yolo/ehs_merged_dataset_v3",
  "class_ids": [1, 2, 3, 4]     // 사용자가 선택한 순서 그대로 넘기면 됨 (순서 = YOLO 라벨 인덱스로 고정 저장)
}
→ { "id": 3, "name": "A공장 안전모니터링", "description": "...", "dataset_path": "...", "status": "active" }
```

`dataset_path`는 실제 이미지/프레임이 올라가는 서버의 절대경로. 프론트에서 경로 입력 필드나
파일 탐색기로 GPU 서버 쪽 경로를 알려줘야 함 (자유 텍스트 입력이어도 무방).

### 프로젝트 목록

```
GET /projects
→ [ { "id": 3, "name": "A공장 안전모니터링", "status": "active" }, ... ]
```

### 프로젝트 상세

```
GET /projects/{project_id}
→ { "id": 3, "name": "...", "description": "...", "dataset_path": "...", "status": "active" }
```

### 프로젝트가 선택한 클래스 목록 (순서 보장됨)

```
GET /projects/{project_id}/classes
→ [ { "id": 1, "name": "forklift", "display_name": "지게차" }, ... ]  // class_index 순서대로
```

### 프로젝트 삭제

```
DELETE /projects/{project_id}
→ { "status": "deleted", "project_id": 3 }
```

---

## 3. 자동라벨링 (대시보드 KPI 카드에 쓰임)

### 라벨링 KPI 조회

```
GET /labeling/kpi?project_id=3
→ {
  "total_target_files": 50,
  "total_target_frames": 15000,
  "processed_files": 32,
  "processed_frames": 9600,
  "saved_frames": 8100,
  "current_accuracy": 0.91   // null일 수 있음 (아직 학습 전)
}
```

### 대상 영상 목록

```
GET /autolabel/videos?project_id=3
→ [ { "id": 10, "filename": "cam1_0709.mp4", "status": "completed",
      "total_frames": 500, "processed_frames": 500, "saved_frames": 420 }, ... ]
```

### 라벨링 시작 (사용자가 영상 선택 후 "라벨링 시작" 버튼)

```
POST /autolabel/start
body: { "project_id": 3, "video_ids": [10, 11, 12] }
→ { "project_id": 3, "class_names": ["forklift","person","helmet","no_helmet"], "video_count": 3, "status": "started" }
```

---

## 4. 학습 (대시보드 메인 기능)

### 학습 시작 ("학습 새로 돌려줘" 버튼)

```
POST /training/run
body: { "project_id": 3, "epochs": 100 }
→ { "run_id": 7, "status": "started", "run_folder": "7_20260709_1530", "base_weights": "..." }
```

`base_weights`는 안 넘겨도 됨 — 백엔드가 자동으로 1차(pretrained)/2차 이후(이전 best.pt)를 판단함.
이미 학습 중이면 409 에러 반환.

### 학습 이력 목록 (좌측 리스트)

```
GET /training/history?project_id=3
→ [ { "id": 7, "version": 2, "started_at": "...", "finished_at": "...",
      "status": "completed", "final_map50": 0.91, "final_map50_95": 0.76,
      "holdout_accuracy": 0.91 }, ... ]
```

`status`는 `running` / `completed` / `failed`. `running`이면 진행 중 표시(progress bar 등)에 활용.

### 학습 상세 (목록에서 클릭 시)

```
GET /training/{run_id}
→ {
  "id": 7, "project_id": 3, "version": 2, "run_folder": "...",
  "status": "completed", "current_epoch": 100, "total_epochs": 100,
  "weights_path": "...", "final_map50": 0.91, "final_map50_95": 0.76,
  "holdout_accuracy": 0.91,
  "started_at": "...", "finished_at": "...",
  "epoch_history": [
    { "epoch": 1, "map50": 0.32, "map50_95": 0.18, "precision": 0.4, "recall": 0.3,
      "train_box_loss": 1.2, "val_box_loss": 1.4 },
    ...
  ],
  "full_report": { ... }   // results.json 전체 내용
}
```

`epoch_history` 배열로 학습 곡선(라인 차트) 그리면 됨.

### 모델 버전 목록 (다운로드 드롭다운용)

```
GET /projects/{project_id}/models
→ [
  { "run_id": 6, "version": 1, "filename": "best_v1.pt",
    "holdout_accuracy": 0.87, "final_map50": 0.85, "finished_at": "...",
    "download_url": "/models/6/pt" },
  { "run_id": 7, "version": 2, "filename": "best_v2.pt",
    "holdout_accuracy": 0.91, "final_map50": 0.89, "finished_at": "...",
    "download_url": "/models/7/pt" }
]
```

`completed` 상태인 학습만 나옴 (진행 중/실패는 제외).

### 버전 선택 후 다운로드 (파일 바이너리 응답)

```
GET /projects/{project_id}/models/{version}/pt
→ best_v{version}.pt 파일 다운로드 (Content-Type: application/octet-stream)
```

프론트에서 `fetch` → `blob()` → `<a download>` 클릭 트리거하는 방식 권장.
예시 코드: `frontend-example/ModelVersionDownloader.jsx` 참고.

---

## 5. 채팅

```
POST /chat
body: { "message": "라벨링 현황 어때?", "project_id": 3 }
→ { "reply": "현재 총 50개 영상 중 32개 처리됐고..." }
```

`project_id`는 현재 사용자가 보고 있는 프로젝트를 채팅 맥락에 알려주기 위한 것. 안 넘기면
Claude가 어떤 프로젝트인지 되물을 수 있음 — 프론트에서 항상 현재 선택된 프로젝트 id를 같이 보내는 걸 권장.

---

## 프론트에서 신경 써야 할 것

1. **CORS**: 백엔드가 `allow_origins=["*"]`로 열려있지만, 운영 배포 시 Electron 앱 origin으로 좁힐 수 있음 — 연동 안 되면 백엔드팀에 확인
2. **폴링 vs 실시간**: 학습 진행상황(`current_epoch` 등)은 WebSocket이 아니라 폴링 방식. 학습 중인 화면에서는 `GET /training/{id}`를 5~10초 간격으로 재호출해서 갱신하는 걸 추천
3. **project_id는 항상 쿼리파라미터 또는 body로 명시**: 대부분의 엔드포인트가 프로젝트 단위로 데이터를 나누기 때문에 빠뜨리면 400/404 남
4. **날짜 포맷**: 모든 timestamp는 ISO 8601 (`2026-07-09T15:30:00+09:00`), `new Date(str)`로 바로 파싱 가능
5. **버전(version)은 null일 수 있음**: 학습이 아직 진행 중이거나 실패했으면 `version`이 없음(아직 부여 안 됨) — 목록 렌더링 시 `run #7` 처럼 fallback 표시 권장
