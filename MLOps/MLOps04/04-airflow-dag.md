# 서브퀘스트 04 (Airflow) — DAG 두 노드 성공 확인

**결론: ✅ 완료**

> 요청 항목: "Airflow DAG 두 노드가 성공 로그로 끝났는지"

## DAG 구성

- DAG id: `bigquery_to_huggingface` (`dags/bigquery_to_huggingface.py`, TaskFlow API)
- 노드 2개:
  1. `bigquery_to_gcs` — BigQuery 테이블을 GCS로 extract
  2. `register_dataset_to_huggingface` — GCS CSV를 HuggingFace 데이터셋으로 업로드

## 최신 실행 결과

- 실행: `run_id=scheduled__2026-06-25T07:52:19.537111+00:00` (가장 최근 run)

| 노드 | 상태 | 로그 근거 |
|---|---|---|
| `bigquery_to_gcs` | ✅ success | `Task instance in success state`, 반환값 `gs://mlops_airflow_midi3008/dataset__15af55da-..._*.csv` |
| `register_dataset_to_huggingface` | ✅ success | `Task instance in success state`, `Done. Returned value was: None` (업로드 완료) |

두 노드 모두 `try_number=1`에서 `RUNNING → success`로 정상 종료.
(과거 manual run들도 다수 존재하나, 최신 scheduled run에서 두 노드 모두 성공 확인.)

## 판정

두 노드 모두 성공 상태 로그로 종료 → **완료**.
