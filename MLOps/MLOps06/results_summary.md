# MLflow Experiment Tracking — 결과 요약

- 실습: MLOps05 — MLflow 로컬 실험 추적 (강의 원본 `mlops-quicklab/mlflow` 재현)
- 실행 방식: `docker compose up -d --build` → MLflow 서버(localhost:5000) + trainer 5개 자동 학습
- 모델/데이터: MNIST 분류, 2-layer MLP (PyTorch), `mlflow.pytorch.autolog()`
- Experiment: **Default** (experiment_id=0), Run 5개 · 전부 **FINISHED**
- 캡처 시각 기준: run 5개 5에폭 학습 완료 후 REST API 조회

## 1. 하이퍼파라미터 스윕 결과 (accuracy 내림차순)

| 순위 | run_name | run_id | batch_size | nn_dim_hidden | lr | epochs | accuracy(%) | loss | status |
|---|---|---|---|---|---|---|---|---|---|
| 1 | industrious-bat-518 | 69bd1908 | 16  | 196 | 0.01 | 5 | **94.548** | 0.1928 | FINISHED |
| 2 | wistful-hog-329     | 970d800e | 32  | 512 | 0.01 | 5 | 92.758 | 0.2589 | FINISHED |
| 3 | honorable-skink-987 | 17952bac | 64  | 128 | 0.01 | 5 | 90.737 | 0.3284 | FINISHED |
| 4 | peaceful-bass-944   | 4d78e27a | 128 | 256 | 0.01 | 5 | 89.127 | 0.4021 | FINISHED |
| 5 | resilient-finch-52  | 4359780e | 128 | 64  | 0.01 | 5 | 88.908 | 0.4121 | FINISHED |

> accuracy/loss는 마지막 에폭(step=4) 기준 값.

## 2. 최고 성능 run

**run `69bd1908` (industrious-bat-518) — accuracy 94.548%, loss 0.1928.**
하이퍼파라미터: `batch_size=16, nn_dim_hidden=196, lr=0.01, epochs=5`.
경향: batch_size가 작을수록(16 → 128) accuracy가 높았고, 배치 128에서는 hidden을 키워도(256) 큰 개선이 없었다.

## 3. Model Registry

| 항목 | 값 |
|---|---|
| 등록 모델(registered model) | `model` |
| 모델 버전(model version) | **v1** (status=READY) |
| 스테이지(Production 여부) | **Production** ✅ |
| 연결된 run_id | `e3d870e5` |

**설명:** compose(`trainer.py` 5개)는 스윕만 하고 모델 등록은 하지 않는다. 그래서 별도로
`docker compose run --rm trainer1 python train_and_register.py`를 1회 실행했다. 이 스크립트는
학습 후 `mlflow.pytorch.log_model` → `create_registered_model("model")` →
`create_model_version` → `transition_model_version_stage("model", 1, "Production")` 순으로
레지스트리에 **`model` v1을 등록하고 Production 스테이지로 승격**한다.
(과거에 있던 빈 placeholder `train_and_register.py` 항목과는 별개의 정식 모델이다.)

## 참고 (강의 repo 특성)

- 5개 run의 `lr`이 모두 0.01로 동일하다. `docker-compose.yaml`은 `LEARNING_RATE` 환경변수를
  주는데, `trainer.py`의 설정 필드명이 `lr`이라 pydantic-settings가 `LR`을 찾아 매칭에 실패한다
  (`BATCH_SIZE`/`NN_DIM_HIDDEN`은 필드명과 일치해 정상 반영). 비교는 batch_size·hidden 차이로 성립한다.
- MNIST 원본 URL(yann.lecun.com)이 404라 학습 시 `ossci-datasets` S3 미러로 자동 폴백한다(정상).
