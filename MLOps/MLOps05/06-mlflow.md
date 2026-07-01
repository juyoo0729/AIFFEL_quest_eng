# 서브퀘스트 — MLflow (mlops-quicklab/mlflow)

**결론: ✅ 완료 — 로컬 실행 + 스윕 로깅 + 모델 레지스트리 Production 등록까지 검증**

> 요청 항목: "MLflow까지" — MLflow 트래킹 + 모델 학습/로깅/레지스트리 등록
> 증거 파일: [`~/mlops/mlflow/results_summary.md`](../mlflow/results_summary.md) (실행 결과를 파일로 캡처)

## 실행 방식

강의 원본 repo(`github.com/KennethanCeyer/mlops-quicklab`)를 새로 clone(`~/mlops/mlflow-lecture/mlflow`)해
`docker compose up -d --build` 로 기동. MLflow 서버(localhost:5000, HTTP 200) + trainer 5개가
서로 다른 하이퍼파라미터로 MNIST MLP를 학습하며 동일 서버에 로깅 → 대시보드에서 run 비교.

| 파일 | 역할 | 상태 |
|---|---|---|
| `trainer.py` | MNIST MLP 학습 + `mlflow.pytorch.autolog()` + loss/accuracy 로깅 | ✅ 실행됨 |
| `train_and_register.py` | 학습 → 모델 로깅(signature) → Model Registry 등록 → Production 전환 | ✅ 실행됨 |
| `docker-compose.yaml` | MLflow UI(:5000) + 5개 trainer(하이퍼파라미터 스윕) | ✅ 실행됨 |

## 1. 하이퍼파라미터 스윕 (5 run, 전부 FINISHED)

REST API(`/api/2.0/mlflow/runs/search`)로 조회한 최종 결과 (accuracy 내림차순):

| run_name | batch_size | nn_dim_hidden | lr | epochs | accuracy(%) | loss |
|---|---|---|---|---|---|---|
| industrious-bat-518 | 16  | 196 | 0.01 | 5 | **94.548** | 0.1928 |
| wistful-hog-329     | 32  | 512 | 0.01 | 5 | 92.758 | 0.2589 |
| honorable-skink-987 | 64  | 128 | 0.01 | 5 | 90.737 | 0.3284 |
| peaceful-bass-944   | 128 | 256 | 0.01 | 5 | 89.127 | 0.4021 |
| resilient-finch-52  | 128 | 64  | 0.01 | 5 | 88.908 | 0.4121 |

**최고 성능:** `industrious-bat-518` — accuracy 94.548%, `batch_size=16, nn_dim_hidden=196`.
경향: batch_size가 작을수록 accuracy가 높았다.

## 2. Model Registry (train_and_register.py)

`docker compose run --rm trainer1 python train_and_register.py` 1회 실행으로 등록:

| 항목 | 값 |
|---|---|
| 등록 모델 | `model` |
| 버전 | **v1** (status=READY) |
| 스테이지 | **Production** ✅ |
| 연결 run_id | `e3d870e5` |

흐름: `log_model(signature)` → `pyfunc.load_model + predict`(검증) →
`create_registered_model("model")` → `create_model_version` →
`transition_model_version_stage("model", 1, "Production")`.

## 3. 디버깅/추가 관찰

- **lr 미반영 이슈:** 5개 run의 `lr`이 모두 0.01로 동일했다. 원인은 `docker-compose.yaml`이
  `LEARNING_RATE` 환경변수를 주는데 `trainer.py`의 pydantic 설정 필드명이 `lr`이라 `LR`을 찾아
  매칭 실패(batch_size·nn_dim_hidden은 필드명과 일치해 정상 반영). 비교는 batch/hidden 차이로 성립.
- **MNIST 404 → 미러 폴백:** 원본 URL(yann.lecun.com)이 404라 학습 시 `ossci-datasets` S3
  미러로 자동 폴백(정상 동작).
- **휘발성 백엔드:** 강의 compose의 MLflow 서비스는 볼륨 마운트가 없어 컨테이너를 내리면 기록이
  사라진다. → 컨테이너 teardown 전에 REST API로 결과를 `results_summary.md`에 캡처해 증거로 남겼다.

## 4. 추가 실험 (표준 로컬 셋업)

강의 원본은 무거운 MNIST+PyTorch 방식이라, 별도로 `~/mlops/mlflow-local`에 경량 표준 셋업도 구성:
MLflow 서버만 Docker로(아티팩트 프록시 `--serve-artifacts`, sqlite 백엔드) 띄우고 격리 venv에서
sklearn RandomForest(wine) 하이퍼파라미터 스윕 5 run을 로깅 → 대시보드 비교 확인.

## 회고

- MLflow의 실험 추적은 "run = 한 번의 학습" 단위로 파라미터·메트릭·모델(아티팩트)을 자동 기록하고,
  autolog만으로도 대부분의 로깅이 잡힌다는 걸 실제 스윕으로 확인했다.
- 가장 값진 교훈은 **증거의 영속화**였다. compose의 MLflow 백엔드가 휘발성이라, 컨테이너를 내리면
  대시보드가 사라진다. 그래서 teardown 전에 REST API로 run·레지스트리 상태를 파일로 떠서 남겼다.
- 환경변수 이름(`LEARNING_RATE`)과 설정 필드명(`lr`)의 불일치처럼, "돌아가긴 하는데 의도대로
  반영 안 되는" 조용한 버그를 run 비교표에서 잡아낼 수 있었다(모든 lr이 동일 → 이상 감지).
