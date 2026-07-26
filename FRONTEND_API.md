# Watchers Frontend 연동 API 문서 / Frontend Integration API Reference

[🇰🇷 한국어](#한국어) ・ [🇺🇸 English](#english)

---

<br>

# 한국어

프론트(Electron + React) 팀이 백엔드와 연동할 때 참고하는 문서. Base URL은 백엔드 서버 주소
(`.env`의 `VITE_API_BASE_URL`, 예: `http://<백엔드IP>:8000`).

모든 응답은 JSON(파일 다운로드 제외). 에러는 `{"detail": "에러 메시지"}` 형태(FastAPI 기본 포맷), 상태 코드는 4xx/5xx.

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
  "class_ids": [1, 2, 3, 4],        // 선택 순서 그대로 넘기면 됨 — 이 순서가 YOLO 라벨 인덱스(class_index)로 영구 고정됨
  "source_model_run_id": null       // (선택) 기존 학습 run id를 넘기면 그 모델이 속한 프로젝트의 학습 이력 전체를 복제해서 시작
}
→ { "id": 3, "name": "A공장 안전모니터링", "description": "...", "dataset_path": "...",
    "status": "idle", "latest_map50": null }
```

- `dataset_path`는 입력받지 않음 — 서버가 `DATASET_ROOT/project_{id}`로 자동 생성해서 응답에만 넣어줌(표시용).
- `status`는 저장된 값이 아니라 **매 요청마다 학습 이력에서 파생**됨: 가장 최근 `TrainingRun`이 없으면 `idle`, 있으면 그 run의 상태(`running`/`completed`/`failed`)를 그대로 반환.
- `latest_map50`은 가장 최근에 **완료된** 학습의 mAP50 (완료된 학습이 하나도 없으면 `null`).
- `source_model_run_id`를 넘기는 경우, **`class_ids`의 순서가 원본 프로젝트의 클래스 순서와 정확히 일치**해야 함(YOLO가 클래스 인덱스를 가중치에 그대로 굽기 때문). 불일치 시 400과 함께 두 순서를 모두 보여주는 에러 메시지가 옴.

### 프로젝트 목록 / 상세

```
GET /projects
→ [ { "id": 3, "name": "A공장 안전모니터링", "status": "completed", "latest_map50": 0.91 }, ... ]

GET /projects/{project_id}
→ { "id": 3, "name": "...", "description": "...", "dataset_path": "...",
    "status": "completed", "latest_map50": 0.91 }
```

### 프로젝트가 선택한 클래스 목록 (순서 보장됨)

```
GET /projects/{project_id}/classes
→ [ { "id": 1, "name": "forklift", "display_name": "지게차" }, ... ]   // class_index 순서대로
```

### 프로젝트 삭제

```
DELETE /projects/{project_id}
→ { "status": "deleted", "project_id": 3 }
```

삭제 시 DB의 학습 이력/epoch/자동라벨링 영상 목록과, 서버에 저장된 데이터셋(이미지/라벨)·
학습 결과 파일(best.pt/results.json)이 전부 같이 삭제됨. **되돌릴 수 없으니 프론트에서 확인 다이얼로그 필수.**

---

## 3. 자동라벨링

### 라벨링 KPI 조회 (대시보드/라벨링현황 화면 KPI 카드)

```
GET /labeling/kpi?project_id=3
→ {
  "total_target_files": 50,
  "total_target_frames": 15000,
  "processed_files": 32,
  "processed_frames": 9600,
  "saved_frames": 8100,
  "current_accuracy": 0.91,   // null 가능 (아직 학습 전)
  "avg_fps": 41.8             // 최근 완료된 영상 최대 10개 기준 평균, null 가능
}
```

### 대상 영상 목록

```
GET /autolabel/videos?project_id=3
→ [ { "id": 10, "filename": "cam1_0709.mp4", "status": "completed",
      "total_frames": 500, "processed_frames": 500, "saved_frames": 420,
      "fps": 29.97, "width": 1920, "height": 1080, "duration_sec": 16.7, "readable": true }, ... ]
```

### 영상 업로드 (multipart, 여러 개 동시 가능)

```
POST /autolabel/videos/upload?project_id=3
form-data: file (여러 개면 file 필드를 반복)
→ [ { "id": 10, "filename": "cam1_0709.mp4", "status": "pending", ... }, ... ]
```

내부적으로 각 파일을 GPU 서버로 그대로 릴레이 업로드함(backend는 영상 바이트를 직접 저장하지 않음).

### 라벨링 시작

```
POST /autolabel/start
body: { "project_id": 3, "video_ids": [10, 11, 12] }
→ { "project_id": 3, "class_names": ["forklift","person","helmet","no_helmet"], "video_count": 3, "status": "started" }
```

### 자동라벨링 실행(run) 이력 — CVAT 검수 루프에 사용

```
GET /autolabel/runs?project_id=3
→ [ { "id": 5, "project_id": 3, "run_key": "project3_20260709_161542", "status": "completed",
      "extracted_frame_count": 420, "dino_box_count": 180, "active_box_count": 150,
      "merged_box_count": 260, "train_image_count": 380, "val_image_count": 40,
      "cvat_image_count": 420, "reviewed_at": null, "started_at": "...", "finished_at": "..." }, ... ]

GET /autolabel/runs/{run_id}
→ (위와 동일한 형태의 상세)
```

### 검수용 CVAT zip 다운로드 (파일 응답)

```
GET /autolabel/runs/{run_id}/cvat-zip
→ cvat_review_yolo11.zip 다운로드 (아직 검수 안 된 이미지만 포함, 자동 라벨이 미리 채워져 있음)
```

### 검수 완료 zip 재업로드 (사람이 CVAT에서 검수 후)

```
POST /autolabel/runs/{run_id}/reviewed-dataset
form-data: file (CVAT에서 export한 YOLO 1.1 zip)
→ { "id": 5, "reviewed_at": "...", "review_zip_path": "...",
    "train_image_count": 420, "val_image_count": 40, "cvat_zip_path": "..." (남은 미검수분) }
```

업로드 즉시 데이터셋이 재생성되므로, 이 응답을 받은 직후 바로 "학습 시작" 버튼을 눌러도 됨.

---

## 4. 학습

### 학습 시작

```
POST /training/run
body: { "project_id": 3, "epochs": 100 }   // imgsz/batch/patience/device/base_weights/note도 선택적으로 지정 가능
→ { "run_id": 7, "status": "started", "run_folder": "7_20260709_1530", "base_weights": "..." }
```

`base_weights`는 안 넘겨도 됨 — 백엔드가 자동으로 승격된 모델(있으면) → 없으면 1차(pretrained) 학습으로 판단함.
GPU 서버가 이미 학습 중이면 409 에러 반환.

### 학습 이력 목록

```
GET /training/history?project_id=3
→ [ { "id": 7, "version": 2, "started_at": "...", "finished_at": "...",
      "status": "completed", "final_map50": 0.91, "final_map50_95": 0.76,
      "holdout_accuracy": 0.91, "promoted": true,
      "current_epoch": 100, "total_epochs": 100 }, ... ]
```

`status`는 `running` / `completed` / `failed`. `promoted`가 `true`인 run이 현재 프로젝트에서 실제 사용 중인 "활성 모델"(프로젝트당 최대 1개).

### 학습 상세 (목록에서 클릭 시)

```
GET /training/{run_id}
→ {
  "id": 7, "project_id": 3, "version": 2, "run_folder": "...",
  "status": "completed", "current_epoch": 100, "total_epochs": 100,
  "weights_path": "...", "final_precision": 0.93, "final_recall": 0.88,
  "final_map50": 0.91, "final_map50_95": 0.76, "holdout_accuracy": 0.91,
  "promoted": true, "promotion_reason": "no_helmet recall 개선 (0.704 -> 0.741), 다른 클래스 안정적",
  "class_metrics": {
    "no_helmet": { "precision": 0.9, "recall": 0.74, "map50": 0.88 }, ...
  },
  "started_at": "...", "finished_at": "...",
  "epoch_history": [
    { "epoch": 1, "map50": 0.32, "map50_95": 0.18, "precision": 0.4, "recall": 0.3,
      "train_box_loss": 1.2, "val_box_loss": 1.4 },
    ...
  ],
  "full_report": { ... }   // results.json 전체 내용, 없으면 null
}
```

`epoch_history` 배열로 학습 곡선(라인 차트)을, `class_metrics`로 클래스별 막대 차트를 그리면 됨.

### 모델 버전 목록 (다운로드 드롭다운용)

```
GET /projects/{project_id}/models
→ [
  { "run_id": 6, "version": 1, "filename": "best_v1.pt", "promoted": false,
    "holdout_accuracy": 0.87, "final_map50": 0.85, "finished_at": "...",
    "download_url": "/models/6/pt" },
  { "run_id": 7, "version": 2, "filename": "best_v2.pt", "promoted": true,
    "holdout_accuracy": 0.91, "final_map50": 0.89, "finished_at": "...",
    "download_url": "/models/7/pt" }
]
```

`completed` + 버전이 부여된 학습만 나옴(진행 중/실패는 제외).

```
GET /models                                    // 전체 프로젝트 통합 목록 (새 프로젝트를 기존 모델에서 시작할 때 선택용)
→ [ { "run_id": 7, "project_id": 3, "project_name": "A공장 안전모니터링", "version": 2, ... }, ... ]
```

### 버전 선택 후 다운로드 (파일 바이너리 응답)

```
GET /projects/{project_id}/models/{version}/pt   // 버전 번호로
GET /models/{run_id}/pt                          // run_id로
→ best_v{version}.pt 다운로드 (Content-Type: application/octet-stream)
```

프론트에서 `fetch` → `blob()` → `<a download>` 클릭 트리거하는 방식 권장.

---

## 5. Predict — 데모/시각화 전용 추론 ("모델 테스트" 화면)

학습 데이터셋과 완전히 무관한 1회성 추론. 채팅의 "이 영상 라벨링 해줘"도 내부적으로 이 API를 사용함.

### 추론 시작

```
POST /predict/run
body: { "project_id": 3, "video_id": 10, "class_conf_overrides": { "no_helmet": 0.05 } }  // overrides는 선택
→ { "id": 21, "project_id": 3, "video_id": 10, "status": "queued", "started_at": "..." }
```

GPU 서버가 이미 predict/자동라벨링 중이면(둘이 GPU device 락을 공유함) 409 반환.

### 진행상황/결과 조회 (폴링)

```
GET /predict/runs/{run_id}
→ {
  "id": 21, "status": "processing",     // queued | processing | completed | failed
  "frame_count": 210, "total_frames": 500, "detection_count": 340,
  "per_class_count": { "forklift": 12, "person": 40, "helmet": 38, "no_helmet": 2 },
  "progress_percent": 42.0, "processing_fps": 38.5,
  "started_at": "...", "finished_at": null, "error": null
}
```

`status`가 `completed`가 되면 결과 영상을 다운로드할 수 있음.

### 결과 영상 다운로드 (파일 바이너리 응답)

```
GET /predict/runs/{run_id}/video
→ 바운딩 박스가 그려진 결과 mp4 다운로드
```

### 프로젝트의 predict 실행 이력

```
GET /predict/runs?project_id=3
→ [ { "id": 21, "video_id": 10, "status": "completed", ... }, ... ]
```

---

## 6. 채팅

```
POST /chat
body: { "message": "라벨링 현황 어때?", "project_id": 3 }
→ { "reply": "현재 총 50개 영상 중 32개 처리됐고..." }
```

채팅으로 "학습 새로 돌려줘" / "라벨링도 같이 시작해줘"처럼 실제 작업을 시킬 수도 있음 (MCP tool을 통해
`/training/run`, `/autolabel/start`, `/predict/run`을 실제로 호출함). 학습(`TRAIN_DEVICE`)과 라벨링(`LABEL_DEVICE`)은 GPU
서버에서 device가 분리돼 있어서, 같은 채팅에서 둘 다 요청해도 서로 안 기다리고 동시에 돌아감. 채팅 응답
자체는 "시작했음" 확인까지만 기다리는 거라 빠르게 옴 — 학습/라벨링이 끝날 때까지 채팅창이 멈추지 않음.

`project_id`는 현재 사용자가 보고 있는 프로젝트를 채팅 맥락에 알려주기 위한 것. 안 넘기면
모델이 어떤 프로젝트인지 되물을 수 있음 — 프론트에서 항상 현재 선택된 프로젝트 id를 같이 보내는 걸 권장.

---

## 프론트에서 신경 써야 할 것

1. **CORS**: 백엔드가 `allow_origins=["*"]`로 열려있지만, 운영 배포 시 Electron 앱 origin으로 좁힐 수 있음 — 연동 안 되면 백엔드팀에 확인
2. **폴링 vs 실시간**: 학습/라벨링/predict 진행상황은 전부 WebSocket이 아니라 폴링 방식. 진행 중인 화면에서는 `GET /training/{id}` / `GET /predict/runs/{id}`를 4~10초 간격으로 재호출해서 갱신하는 걸 추천(진행 중인 항목이 없으면 폴링을 멈추는 것 권장)
3. **project_id는 항상 쿼리파라미터 또는 body로 명시**: 대부분의 엔드포인트가 프로젝트 단위로 데이터를 나누기 때문에 빠뜨리면 400/404 남
4. **날짜 포맷**: 모든 timestamp는 ISO 8601 (`2026-07-09T15:30:00+09:00`), `new Date(str)`로 바로 파싱 가능
5. **버전(version)은 null일 수 있음**: 학습이 아직 진행 중이거나 실패했으면 `version`이 없음(아직 부여 안 됨) — 목록 렌더링 시 `run #7` 처럼 fallback 표시 권장
6. **`promoted` 필드로 "현재 활성 모델" 표시**: 프로젝트당 최대 1개의 `training_runs` row만 `promoted: true`이며, 이 모델이 다음 자동라벨링의 Active Learning 초안과 predict 추론에 실제로 쓰이는 모델임 — 학습 이력 화면에서 뱃지로 강조 권장
7. **`/autolabel/videos/upload`, `/autolabel/runs/{id}/reviewed-dataset`는 대용량 파일 업로드**: 타임아웃을 넉넉히 잡고, 업로드 진행률 UI(브라우저 `XMLHttpRequest.upload.onprogress` 등)를 고려할 것

---

<br>

# English

Reference for the frontend (Electron + React) team integrating with the backend. Base URL is the
backend server address (`.env`'s `VITE_API_BASE_URL`, e.g. `http://<backend-IP>:8000`).

All responses are JSON except file downloads. Errors come back as `{"detail": "error message"}`
(FastAPI's default shape), with a 4xx/5xx status code.

---

## 0. Health Check

```
GET /health
→ { "status": "ok" }
```

---

## 1. Classes

### List classes (for the project-creation checkbox screen)

```
GET /classes
→ [ { "id": 1, "name": "forklift", "display_name": "지게차" }, ... ]
```

### Add a class (admin use — rarely called directly from the frontend)

```
POST /classes
body: { "name": "fire-extinguisher", "display_name": "소화기" }
→ { "id": 5, "name": "fire-extinguisher", "display_name": "소화기" }
```

---

## 2. Projects

### Create a project (user selects classes via checkboxes)

```
POST /projects
body: {
  "name": "Plant A Safety Monitoring",
  "description": "Forklift/worker detection",
  "class_ids": [1, 2, 3, 4],        // pass them in selection order — this order is permanently locked in as the YOLO label index (class_index)
  "source_model_run_id": null       // (optional) pass an existing run id to clone that model's project's entire training history as a starting point
}
→ { "id": 3, "name": "Plant A Safety Monitoring", "description": "...", "dataset_path": "...",
    "status": "idle", "latest_map50": null }
```

- `dataset_path` is not user-supplied — the server generates `DATASET_ROOT/project_{id}` automatically and returns it purely for display.
- `status` isn't a stored value — it's **derived from training history on every request**: `idle` if there's no `TrainingRun` yet, otherwise the most recent run's status (`running`/`completed`/`failed`).
- `latest_map50` is the mAP50 of the most recently **completed** run (`null` if none has completed yet).
- If `source_model_run_id` is provided, **`class_ids` must be in exactly the same order as the source project's classes** (because YOLO bakes the class index into the trained weights). A mismatch returns 400 with both orderings shown in the error message.

### List / detail

```
GET /projects
→ [ { "id": 3, "name": "Plant A Safety Monitoring", "status": "completed", "latest_map50": 0.91 }, ... ]

GET /projects/{project_id}
→ { "id": 3, "name": "...", "description": "...", "dataset_path": "...",
    "status": "completed", "latest_map50": 0.91 }
```

### The project's selected classes (order preserved)

```
GET /projects/{project_id}/classes
→ [ { "id": 1, "name": "forklift", "display_name": "지게차" }, ... ]   // in class_index order
```

### Delete a project

```
DELETE /projects/{project_id}
→ { "status": "deleted", "project_id": 3 }
```

Deletion cascades to training history/epochs/auto-labeling video records in the DB, plus the dataset
(images/labels) and training artifacts (best.pt/results.json) stored on disk. **This is irreversible —
always show a confirmation dialog in the frontend.**

---

## 3. Auto-labeling

### Labeling KPIs (for the dashboard / labeling-status KPI cards)

```
GET /labeling/kpi?project_id=3
→ {
  "total_target_files": 50,
  "total_target_frames": 15000,
  "processed_files": 32,
  "processed_frames": 9600,
  "saved_frames": 8100,
  "current_accuracy": 0.91,   // may be null (before any training)
  "avg_fps": 41.8             // average over the last up-to-10 completed videos; may be null
}
```

### List target videos

```
GET /autolabel/videos?project_id=3
→ [ { "id": 10, "filename": "cam1_0709.mp4", "status": "completed",
      "total_frames": 500, "processed_frames": 500, "saved_frames": 420,
      "fps": 29.97, "width": 1920, "height": 1080, "duration_sec": 16.7, "readable": true }, ... ]
```

### Upload videos (multipart, multiple files at once)

```
POST /autolabel/videos/upload?project_id=3
form-data: file (repeat the `file` field for multiple files)
→ [ { "id": 10, "filename": "cam1_0709.mp4", "status": "pending", ... }, ... ]
```

Each file is relayed straight through to the GPU server internally — the backend never stores video bytes itself.

### Start labeling

```
POST /autolabel/start
body: { "project_id": 3, "video_ids": [10, 11, 12] }
→ { "project_id": 3, "class_names": ["forklift","person","helmet","no_helmet"], "video_count": 3, "status": "started" }
```

### Auto-labeling run history — used for the CVAT review loop

```
GET /autolabel/runs?project_id=3
→ [ { "id": 5, "project_id": 3, "run_key": "project3_20260709_161542", "status": "completed",
      "extracted_frame_count": 420, "dino_box_count": 180, "active_box_count": 150,
      "merged_box_count": 260, "train_image_count": 380, "val_image_count": 40,
      "cvat_image_count": 420, "reviewed_at": null, "started_at": "...", "finished_at": "..." }, ... ]

GET /autolabel/runs/{run_id}
→ (same shape as above, single run detail)
```

### Download the CVAT review package (file response)

```
GET /autolabel/runs/{run_id}/cvat-zip
→ downloads cvat_review_yolo11.zip (only still-unreviewed images, pre-filled with auto labels)
```

### Re-upload the reviewed zip (after a human reviews it in CVAT)

```
POST /autolabel/runs/{run_id}/reviewed-dataset
form-data: file (a YOLO 1.1 zip exported from CVAT)
→ { "id": 5, "reviewed_at": "...", "review_zip_path": "...",
    "train_image_count": 420, "val_image_count": 40, "cvat_zip_path": "..." (remaining unreviewed images) }
```

The dataset is regenerated immediately on upload, so it's safe to enable the "start training" button
as soon as this response comes back.

---

## 4. Training

### Start training

```
POST /training/run
body: { "project_id": 3, "epochs": 100 }   // imgsz/batch/patience/device/base_weights/note are all optional
→ { "run_id": 7, "status": "started", "run_folder": "7_20260709_1530", "base_weights": "..." }
```

`base_weights` can be omitted — the backend automatically resolves it to the promoted model (if any),
falling back to a first-time (pretrained) run. Returns 409 if the GPU server is already training.

### Training history

```
GET /training/history?project_id=3
→ [ { "id": 7, "version": 2, "started_at": "...", "finished_at": "...",
      "status": "completed", "final_map50": 0.91, "final_map50_95": 0.76,
      "holdout_accuracy": 0.91, "promoted": true,
      "current_epoch": 100, "total_epochs": 100 }, ... ]
```

`status` is `running` / `completed` / `failed`. The run with `promoted: true` is the "active model"
currently in use for this project (at most one at a time).

### Training detail (on row click)

```
GET /training/{run_id}
→ {
  "id": 7, "project_id": 3, "version": 2, "run_folder": "...",
  "status": "completed", "current_epoch": 100, "total_epochs": 100,
  "weights_path": "...", "final_precision": 0.93, "final_recall": 0.88,
  "final_map50": 0.91, "final_map50_95": 0.76, "holdout_accuracy": 0.91,
  "promoted": true, "promotion_reason": "no_helmet recall improved (0.704 -> 0.741), other classes stable",
  "class_metrics": {
    "no_helmet": { "precision": 0.9, "recall": 0.74, "map50": 0.88 }, ...
  },
  "started_at": "...", "finished_at": "...",
  "epoch_history": [
    { "epoch": 1, "map50": 0.32, "map50_95": 0.18, "precision": 0.4, "recall": 0.3,
      "train_box_loss": 1.2, "val_box_loss": 1.4 },
    ...
  ],
  "full_report": { ... }   // full contents of results.json, or null
}
```

Use the `epoch_history` array for a training-curve line chart, and `class_metrics` for a per-class bar chart.

### Model version list (for a download dropdown)

```
GET /projects/{project_id}/models
→ [
  { "run_id": 6, "version": 1, "filename": "best_v1.pt", "promoted": false,
    "holdout_accuracy": 0.87, "final_map50": 0.85, "finished_at": "...",
    "download_url": "/models/6/pt" },
  { "run_id": 7, "version": 2, "filename": "best_v2.pt", "promoted": true,
    "holdout_accuracy": 0.91, "final_map50": 0.89, "finished_at": "...",
    "download_url": "/models/7/pt" }
]
```

Only `completed` and versioned runs appear (in-progress/failed are excluded).

```
GET /models                                    // catalog across all projects (for picking a starting model when creating a new project)
→ [ { "run_id": 7, "project_id": 3, "project_name": "Plant A Safety Monitoring", "version": 2, ... }, ... ]
```

### Download a specific version (binary file response)

```
GET /projects/{project_id}/models/{version}/pt   // by version number
GET /models/{run_id}/pt                          // by run_id
→ downloads best_v{version}.pt (Content-Type: application/octet-stream)
```

Recommended pattern on the frontend: `fetch` → `blob()` → trigger a hidden `<a download>` click.

---

## 5. Predict — Demo/visualization-only Inference ("Model Test" screen)

A one-off inference completely independent of the training dataset. Chat's "run detection on this
video" also uses this API internally.

### Start inference

```
POST /predict/run
body: { "project_id": 3, "video_id": 10, "class_conf_overrides": { "no_helmet": 0.05 } }  // overrides optional
→ { "id": 21, "project_id": 3, "video_id": 10, "status": "queued", "started_at": "..." }
```

Returns 409 if the GPU server is already busy with predict or auto-labeling (they share a GPU device lock).

### Poll for progress/result

```
GET /predict/runs/{run_id}
→ {
  "id": 21, "status": "processing",     // queued | processing | completed | failed
  "frame_count": 210, "total_frames": 500, "detection_count": 340,
  "per_class_count": { "forklift": 12, "person": 40, "helmet": 38, "no_helmet": 2 },
  "progress_percent": 42.0, "processing_fps": 38.5,
  "started_at": "...", "finished_at": null, "error": null
}
```

Once `status` becomes `completed`, the result video is available for download.

### Download the result video (binary file response)

```
GET /predict/runs/{run_id}/video
→ downloads an mp4 with bounding boxes drawn
```

### A project's predict run history

```
GET /predict/runs?project_id=3
→ [ { "id": 21, "video_id": 10, "status": "completed", ... }, ... ]
```

---

## 6. Chat

```
POST /chat
body: { "message": "How's labeling going?", "project_id": 3 }
→ { "reply": "Out of 50 videos, 32 have been processed so far..." }
```

Chat can also trigger real work — "kick off a new training run" / "start labeling too" actually call
`/training/run`, `/autolabel/start`, `/predict/run` via MCP tools. Since training (`TRAIN_DEVICE`) and
labeling (`LABEL_DEVICE`) use separate GPU devices, requesting both in the same chat runs them
concurrently rather than one waiting on the other. The chat response itself only waits for the
"started" confirmation, so it returns quickly — the chat window never blocks until training/labeling
finishes.

`project_id` tells the chat which project the user is currently viewing. If omitted, the model may
ask which project you mean — the frontend should always send the currently selected project's id.

---

## Things the Frontend Needs to Handle

1. **CORS**: the backend currently allows `allow_origins=["*"]`, though this may be narrowed to the Electron app's origin in production — check with the backend team if integration breaks.
2. **Polling, not real-time**: training/labeling/predict progress is all poll-based, not WebSocket. Re-call `GET /training/{id}` / `GET /predict/runs/{id}` every 4–10 seconds while a screen shows an in-progress job (and stop polling once nothing is in progress).
3. **Always pass `project_id`** as a query param or in the body: most endpoints scope data per project, and omitting it leaves you with a 400/404.
4. **Date format**: every timestamp is ISO 8601 (`2026-07-09T15:30:00+09:00`), directly parseable with `new Date(str)`.
5. **`version` can be null**: a run that's still in progress or failed has no `version` assigned yet — render a fallback like `run #7` in lists.
6. **Use the `promoted` field to mark the "active model"**: at most one `training_runs` row per project has `promoted: true`, and that's the model actually used for the next round of Active-Learning auto-labeling and for predict inference — worth a badge in the training-history UI.
7. **`/autolabel/videos/upload` and `/autolabel/runs/{id}/reviewed-dataset` are large-file uploads**: use generous timeouts and consider an upload-progress UI (e.g. `XMLHttpRequest.upload.onprogress`).
