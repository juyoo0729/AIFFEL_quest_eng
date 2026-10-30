# AIFFEL 학습·실습 기록 | 유정욱

AIFFEL 과정에서 진행한 Python, 데이터 분석, 컴퓨터 비전, NLP, LLM·RAG, MLOps 실습을 정리한 저장소입니다.
교육 자료를 바탕으로 한 실습과 피어리뷰, 실행 결과 및 문제 해결 기록을 함께 보관합니다.

[GitHub 프로필](https://github.com/juyoo0729) · [기술 블로그](https://juyoo0729.tistory.com/)

## 먼저 살펴볼 기록

| 분야 | 파일 | 확인할 내용 |
| --- | --- | --- |
| RAG | [RAG 기초 실습](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/LLM_Application/LLM01/Day1_RAG.ipynb) | 검색과 답변 생성 흐름을 다룬 노트북 |
| RAG 평가 | [RAGAS 평가 노트북](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/LLM_Application/LLM02/ragas_eval.ipynb) | 평가 코드와 저장된 출력 |
| 모델 서빙 | [GCP 서빙 기록](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/MLOps/MLOps03/03-gcp-serving.md) · [예측 응답](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/MLOps/MLOps03/predict_response.json) | Vertex AI 학습·서빙·응답 캡처 과정 |
| 실험 관리 | [MLflow 실습 결과](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/MLOps/MLOps06/results_summary.md) | 5개 실행의 설정·학습 지표와 모델 등록 기록 |
| 파이프라인 | [Airflow DAG 기록](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/MLOps/MLOps04/04-airflow-dag.md) | 학습 작업 흐름 실습 |
| 컴퓨터 비전 | [CV01](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/Computer_Vision/CV01/cv01.ipynb) · [CV02](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/Computer_Vision/CV02/CV2.ipynb) | 모델 실습 노트북과 출력 |
| 데이터 분석 | [Spaceship Titanic 실습](https://github.com/juyoo0729/AIFFEL_quest_eng/blob/main/Main_Quest/Quest01/Spaceship%20Titanic.ipynb) | 데이터 탐색·예측 실습 |

## 학습 내용을 프로젝트에 연결한 경험

- **[Project Han](https://juyoo0729.tistory.com/29):** 야간 영상의 차선 클래스 불균형을 분석하고 U-Net의 가중 손실 비교와 후처리를 수행했습니다.
- **[ClaFact](https://github.com/juyoo0729/clafact-auto):** 공공 통계의 구조를 확인하고 뉴스 수치에 대응하는 KOSIS 통계표·항목·시점을 탐색했습니다.
- **[제주 지니](https://github.com/juyoo0729/jeju-genie):** 공공데이터의 REST/LINK 유형을 구분하고 팀이 활용할 데이터 목록을 정리했습니다.

위 항목은 교육 프로젝트에서 맡았던 역할입니다. 각 팀 서비스의 전체 구현을 개인 작업으로 표시하지 않습니다.

## 폴더별 안내

| 폴더 | 내용 |
| --- | --- |
| [Python](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/Python) | 파이썬 기초·자료구조·클래스 실습 |
| [Data_Analysis](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/Data_Analysis) | 데이터 분석 과제 |
| [Computer_Vision](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/Computer_Vision) | 컴퓨터 비전 과제 |
| [NLP](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/NLP) | 자연어 처리·Transformer·챗봇 실습 |
| [LLM_Application](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/LLM_Application) | RAG·평가·Agent 프레임워크 실습 |
| [MLOps](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/MLOps) | 모델 서빙·Airflow·MLflow 학습 기록 |
| [Deployment](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/Deployment) | 모델 배포 교육 기록 |
| [Main_Quest](https://github.com/juyoo0729/AIFFEL_quest_eng/tree/main/Main_Quest) | 종합 과제와 피어리뷰 |

## 기록을 읽을 때

- 노트북과 문서는 작성 당시의 교육 환경과 실행 결과를 보존합니다. API 키·패키지 버전·데이터가 필요한 실습은 해당 조건을 확인해야 합니다.
- Vertex AI 서빙 기록의 엔드포인트는 응답 검증 후 비용 관리를 위해 삭제했습니다. 현재 접속 가능한 서비스 주소로 사용할 수 없습니다.
- MLflow 결과는 강의 실습 재현 기록입니다. 마지막 에폭의 지표와 독립 테스트 성능은 구분하며, 레지스트리의 Production 표기는 실제 사용자 대상 운영을 뜻하지 않습니다.
- 후속 보완에서는 실행 환경, 본인 변경 사항, 실패 해결 과정을 각 실습의 README에 추가할 예정입니다.
