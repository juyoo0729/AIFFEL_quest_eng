# 서브퀘스트 03 (GCP) — 서빙 파이프라인 확인

**결론: ✅ 완료 — 엔드포인트 배포 + predict/curl 응답 캡처까지 검증**

> 요청 항목: "학습 → 서빙 → 서빙 엔드포인트 curl(predict) 응답까지"
> 증거 파일: [`~/mlops/netflix/predict_response.json`](../netflix/predict_response.json)
> (요청/응답 JSON + 재현용 curl 예시 저장)
>
> 참고(구현 해석): 요청서 문구의 "노트북/BERT"와 달리, 실제 구현은 **Vertex AI 커스텀
> 트레이닝 잡**으로 학습한 **Netflix Prize 행렬분해 추천모델(MFRecommender)**을 **TorchServe
> 커스텀 컨테이너**로 Vertex 엔드포인트에 서빙하는 **개념적으로 동등한 end-to-end 파이프라인**이다.

## 항목별 확인

| 체크 항목 | 상태 | 근거 |
|---|---|---|
| 학습 | ✅ | Vertex AI 커스텀 트레이닝(`netflix/training/trainer/task.py`) → `model.pth`/`config.json`/`maps.json` GCS 업로드 |
| 모델 아티팩트 | ✅ | GCS `gs://mlops_airflow_midi3008/netflix/model/` + 로컬 `serving/model.pth`(~63MB) 존재 |
| TorchServe 서빙 | ✅ | 커스텀 컨테이너 완비: `serving/Dockerfile`, `serving/handler.py`, `serving/config.properties`. AR 이미지 `netflix-serving:latest` 존재 |
| 엔드포인트 배포 | ✅ | CPU(n1-standard-2) 배포 성공, `deployedModel=7327877762243362816` |
| **predict / curl 응답** | ✅ | **`predict_response.json`에 요청·응답·curl 캡처 완료** (이번에 해결) |

## predict 응답 증거 (핵심)

CPU 엔드포인트(`n1-standard-2`, T4 미사용)에 배포 후 predict 요청 → 응답을 파일로 캡처:

**요청 instances** (`user_id=6` 기존 유저 + unknown user cold-start, `top_k=5`):
```json
{"instances": [{"user_id": 6, "top_k": 5}, {"user_id": 999999999, "top_k": 5}]}
```

**응답 (요약):**
| user_id | known_user | 추천 movie_id (score 내림차순) |
|---|---|---|
| 6 | true | 7230(4.62), 7057(4.57), 3456(4.57), 2102(4.48), 5582(4.43) |
| 999999999 | false (cold-start) | 7230(4.72), 7057(4.70), 3456(4.67), 2102(4.58), 5582(4.53) |

**재현용 curl** (predict_response.json에 저장):
```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  https://asia-northeast3-aiplatform.googleapis.com/v1/projects/829311465833/locations/asia-northeast3/endpoints/<ENDPOINT_ID>:predict \
  -d '{"instances": [{"user_id": 6, "top_k": 5}, {"user_id": 999999999, "top_k": 5}]}'
```
> 응답은 엔드포인트가 LIVE인 동안 캡처했다. 비용 절감을 위해 캡처 직후 엔드포인트/모델을
> undeploy·삭제했으므로 위 `<ENDPOINT_ID>`는 현재 stale(삭제됨)이다. curl은 재현 형태로 보존.

## 실제 파이프라인 흐름

```
preprocess.py (netflix_12_half.npz 생성)
  → training/trainer/task.py  (Vertex AI T4, MF 학습 → model.pth/config/maps GCS 업로드)
  → serving/build_serving.sh  (GCS 아티팩트 pull → 이미지 빌드/푸시, torch-model-archiver로 .mar)
  → deploy_cpu.py             (Vertex 엔드포인트 CPU 배포 — T4 대신 저비용 CPU)
  → predict_capture.py        (predict 요청 → 요청/응답/curl을 predict_response.json에 저장)
  → manage.py cleanup         (undeploy → 엔드포인트 삭제 → 모델 삭제, 과금 중단)
```

## 남은 해석 이슈 (요청서 vs 구현)

요청서 문구와 실제 구현이 다른 지점(정보용, 구현 자체는 정상):
- **노트북(.ipynb) 없음** — 학습은 노트북이 아니라 Vertex 커스텀 트레이닝 잡으로 수행.
- **모델이 BERT가 아님** — Netflix Prize 행렬분해 추천모델(`nn.Embedding` 기반).
- **디스크상 `.mar` 없음** — `.mar`는 서빙 이미지 빌드 타임에 생성되어 이미지 내부에만 존재.
  → 서빙 동작 자체는 predict 응답 캡처로 검증됨.

## 회고

- 서빙의 최종 증거는 결국 "엔드포인트가 실제 요청에 응답하는가"이며, 그걸 파일로 남겨야 재현
  가능한 증거가 된다는 걸 확인했다. 이전엔 predict를 성공시키고도 응답을 저장하지 않아 증거가
  없었는데, 이번에 요청/응답/curl을 JSON으로 캡처해 공백을 메웠다.
- 비용 관점에서, predict 검증에는 GPU가 필요 없다는 점(핸들러가 cuda 미가용 시 cpu 폴백)을 이용해
  T4 대신 CPU로 배포 → 동일한 예측 응답을 얻으면서 비용/쿼터 부담을 없앴다.
- 캡처 → 즉시 정리(undeploy·삭제)로 "증거는 남기고 과금은 끊는" 흐름을 실천했다.
