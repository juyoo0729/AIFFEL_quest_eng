# 모델 배포 개론 02 (Day 2) — 제출

FastAPI 기초와 데이터 처리 실습 수행 내역 및 체크포인트 답변입니다.
모든 API 호출은 로컬에서 서버를 직접 실행하여 검증했습니다 (실행 스크립트: `tests/run_submission.py`).

---

## 1. 섹션 1.5 수행내역 — 최소한의 FastAPI 서버

`app/main_basic.py` 작성 후 uvicorn으로 서버를 실행하고 두 엔드포인트를 호출했습니다.

```
GET /health → 상태 코드: 200
응답: {'status': 'healthy'}

GET / → 상태 코드: 200
응답: {'message': 'ML Model Serving API', 'docs_url': '/docs'}
```

<!-- 여기에 노트북 셀 실행 화면 캡쳐를 첨부하세요 -->
<!-- ![섹션 1.5 수행 캡쳐](images/section1-5.png) -->

---

## 2. 섹션 2, 3 셀 출력

### 섹션 2 — Path / Query / Body 파라미터 (`app/main_params.py`)

**2.2 Path 파라미터**

```
GET /models/sentiment-v1     → {'model_name': 'sentiment-v1', 'status': 'running', 'version': '1.0.0'}
GET /models/image-classifier → {'model_name': 'image-classifier', 'status': 'running', 'version': '1.0.0'}
```

**2.2 타입 지정의 효과 (`prediction_id: int`)**

```
GET /predictions/42  → 상태: 200, 응답: {'prediction_id': 42, 'label': '긍정', 'confidence': 0.92}
GET /predictions/abc → 상태: 422
에러: {"detail": [{"type": "int_parsing", "loc": ["path", "prediction_id"],
       "msg": "Input should be a valid integer, unable to parse string as an integer",
       "input": "abc"}]}
```

**2.3 Query 파라미터**

```
GET /models
→ {"total": 3, "models": [{"name": "sentiment-v1", "status": "running"},
                          {"name": "image-clf-v2", "status": "running"},
                          {"name": "ner-v1", "status": "stopped"}]}

GET /models?status=running
→ {"total": 2, "models": [{"name": "sentiment-v1", "status": "running"},
                          {"name": "image-clf-v2", "status": "running"}]}

GET /models?status=running&limit=1
→ {"total": 1, "models": [{"name": "sentiment-v1", "status": "running"}]}
```

**2.4 Request Body (POST /predict)**

```
json={"text": "이 영화 정말 재밌다"}
→ {"label": "긍정", "confidence": 0.92, "probabilities": null}

json={"text": "이 영화 정말 재밌다", "return_probabilities": true}
→ {"label": "긍정", "confidence": 0.92, "probabilities": {"긍정": 0.92, "부정": 0.05, "중립": 0.03}}
```

**2.4 잘못된 요청 테스트**

```
text 필드 누락   → 상태: 422, 에러: Field required
text에 숫자 전달 → 상태: 422, 에러: Input should be a valid string
```

### 섹션 3 — Swagger UI / OpenAPI 문서

**3.4 자동 생성된 OpenAPI 스펙 (`GET /openapi.json`)**

```
API 제목: Parameter Examples
API 버전: 0.1.0
등록된 엔드포인트:
  GET    /models/{model_name}
  GET    /predictions/{prediction_id}
  GET    /models
  POST   /predict
```

**PredictRequest의 JSON Schema**

```json
{
  "properties": {
    "text": { "type": "string", "title": "Text" },
    "return_probabilities": {
      "type": "boolean",
      "title": "Return Probabilities",
      "default": false
    }
  },
  "type": "object",
  "required": ["text"],
  "title": "PredictRequest"
}
```

**3.6 ReDoc 동작 확인 (`GET /redoc`)**

```
상태: 200
내용 길이: 902
```

**4.5 422 에러 응답의 구조 (text 누락 + 잘못된 타입 전송)**

```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "text"],
      "msg": "Field required",
      "input": { "return_probabilities": "yes" }
    }
  ]
}
```

<!-- 여기에 Swagger UI (/docs) 화면 캡쳐를 첨부하세요 -->
<!-- ![Swagger UI](images/swagger-ui.png) -->

---

## 3. 섹션 5 수행내역 — MNIST 모델 추론 API

`app/schemas.py`(Pydantic 스키마), `app/main.py`(추론 서버)를 작성하고
Day 1에서 저장한 `models/mnist_state_dict.pth`를 연결하여 테스트했습니다.

**테스트 1: 헬스체크**

```
GET /health → 상태 코드: 200
응답: {'status': 'healthy', 'model_loaded': True}
```

**테스트 2: 실제 MNIST 이미지로 추론**

```
이미지 크기: (1, 28, 28)
정답 레이블: 7
픽셀 값 개수: 784

POST /predict → 상태 코드: 200
응답: {
  "label": 7,
  "confidence": 1.0,
  "probabilities": null,
  "model_version": "1.0.0"
}
```

**테스트 3: 확률 분포 포함 요청 (`return_probabilities: true`)**

```
예측: 7 (확신도: 1.0)
클래스별 확률:
  0: 0.0000
  1: 0.0000
  2: 0.0000
  3: 0.0000
  4: 0.0000
  5: 0.0000
  6: 0.0000
  7: 1.0000 ##################################################
  8: 0.0000
  9: 0.0000
```

**테스트 4: 여러 이미지 연속 테스트**

```
이미지   정답   예측   확신도     결과
---------------------------------------------
  #0     7      7      1.0        O
  #1     2      2      1.0        O
  #2     1      1      0.9999     O
  #3     0      0      1.0        O
  #4     4      4      1.0        O
  #5     1      1      1.0        O
  #6     4      4      0.9917     O
  #7     9      9      0.951      O
  #8     5      5      0.72       O
  #9     9      9      0.9998     O
정확도: 10/10 (100%)
```

**테스트 5: 에러 상황 테스트**

```
에러 1 (픽셀 100개만 전송) → 상태: 422, 메시지: List should have at least 784 items after validation, not 100
에러 2 (잘못된 타입)       → 상태: 422, 메시지: Input should be a valid list
에러 3 (필수 필드 누락)    → 상태: 422, 메시지: Field required
에러 4 (빈 JSON)           → 상태: 422, 메시지: Field required
```

모든 에러 상황에서 서버가 죽지 않고 적절한 상태 코드와 메시지를 반환하는 것을 확인했습니다.

<!-- 여기에 섹션 5 수행 화면 캡쳐를 첨부하세요 -->
<!-- ![섹션 5 수행 캡쳐](images/section5.png) -->

---

## 4. 각 섹션 체크포인트 답변

### 섹션 1 — FastAPI 개요

**Q1. FastAPI가 Flask보다 모델 배포에 적합한 이유 세 가지는 무엇입니까?**

1. **자동 데이터 검증**: Pydantic이 내장되어 있어 요청이 모델에 도달하기 전에 타입·범위·필수 여부를 자동으로 검증하고, 실패 시 상세한 422 에러를 반환합니다. Flask는 검증 코드를 직접 작성하거나 별도 라이브러리를 붙여야 합니다.
2. **자동 API 문서화**: 코드만 작성하면 Swagger UI(`/docs`)가 자동 생성되어, 별도 문서 작성 없이 브라우저에서 바로 API를 테스트할 수 있습니다.
3. **비동기 처리**: async/await 기반(ASGI)이라 추론처럼 시간이 걸리는 작업 중에도 동시에 여러 요청을 처리할 수 있습니다. Flask는 동기(WSGI) 기반입니다.

**Q2. Uvicorn의 역할은 무엇이며, 왜 FastAPI와 함께 사용합니까?**

Uvicorn은 ASGI 서버로, 네트워크에서 HTTP 요청을 받아 FastAPI 애플리케이션에 전달하는 "문지기" 역할을 합니다. FastAPI는 프레임워크일 뿐 자체적으로 요청을 수신하는 기능이 없기 때문에, 실제로 포트를 열고 요청을 받아주는 ASGI 서버인 Uvicorn이 반드시 필요합니다.

**Q3. `@app.get("/health")`에서 `get`과 `"/health"`는 각각 무엇을 의미합니까?**

- `get`: 이 엔드포인트가 처리할 **HTTP 메서드**(GET)를 의미합니다.
- `"/health"`: 이 엔드포인트가 처리할 **URL 경로**를 의미합니다.

즉 "GET 메서드로 /health 경로에 요청이 오면 바로 아래 함수를 실행하라"는 선언입니다.

**Q4. FastAPI에서 dict를 반환하면 어떤 일이 자동으로 일어납니까?**

FastAPI가 dict를 자동으로 JSON으로 직렬화하여 응답 본문에 담고, `Content-Type: application/json` 헤더와 함께 반환합니다. `json.dumps()`를 직접 호출할 필요가 없습니다.

### 섹션 2 — Path / Query / Body 파라미터

**Q1. `/models/sentiment-v1`에서 `sentiment-v1`은 어떤 종류의 파라미터입니까?**

**Path 파라미터**입니다. URL 경로의 일부로 전달되며, 특정 리소스(여기서는 특정 모델)를 식별하는 데 사용됩니다.

**Q2. `/models?status=running&limit=5`에서 `status`와 `limit`은 어떤 종류의 파라미터입니까?**

**Query 파라미터**입니다. URL 뒤에 `?key=value&key=value` 형태로 전달되며, 검색·필터링·페이지네이션 같은 선택적 조건에 사용됩니다.

**Q3. 모델 추론 요청에 Request Body를 사용하는 이유는 무엇입니까?**

추론 입력은 텍스트, 픽셀 배열, 옵션 등 **복잡하고 구조화된 데이터**이고 양도 많을 수 있기 때문입니다. URL은 길이 제한이 있지만 Body는 제한이 없고, JSON으로 중첩 구조를 표현할 수 있으며, Pydantic 모델로 스키마를 정의해 자동 검증할 수 있습니다.

**Q4. FastAPI에서 함수의 파라미터가 Path, Query, Body 중 어디서 오는지 어떻게 판별합니까?**

- URL 경로의 `{}` 변수와 이름이 같은 파라미터 → **Path 파라미터**
- 경로에 없는 단순 타입(str, int 등) 파라미터 → **Query 파라미터**
- Pydantic `BaseModel` 타입으로 선언된 파라미터 → **Request Body**

### 섹션 3 — Swagger UI

**Q1. FastAPI에서 Swagger UI에 접속하려면 어떤 URL로 이동합니까?**

`http://localhost:8000/docs` (서버 주소 + `/docs` 경로)

**Q2. Swagger UI가 코드와 항상 동기화될 수 있는 이유는 무엇입니까?**

문서를 사람이 따로 작성하는 것이 아니라, FastAPI가 코드(엔드포인트 선언, 타입 힌트, Pydantic 모델, docstring)에서 **OpenAPI 스펙(`/openapi.json`)을 자동 생성**하고 Swagger UI는 그 스펙을 읽어 화면을 그리기 때문입니다. 코드를 수정하면 스펙이 다시 생성되므로 문서가 어긋날 수 없습니다.

**Q3. Pydantic 모델의 `Field(description=, examples=)`는 Swagger UI의 어디에 반영됩니까?**

해당 필드의 **스키마 설명과 예시 입력값**으로 반영됩니다. `description`은 Request body 스키마의 각 필드 설명에 표시되고, `examples`는 [Try it out] 시 입력란에 미리 채워지는 예시 JSON으로 표시됩니다.

**Q4. Swagger UI와 ReDoc의 핵심 차이는 무엇입니까?**

둘 다 같은 OpenAPI 스펙에서 자동 생성되지만, **Swagger UI(`/docs`)는 [Try it out]으로 브라우저에서 API를 직접 호출·테스트**할 수 있어 개발·테스트에 적합하고, **ReDoc(`/redoc`)은 호출 기능 없이 읽기 전용**으로 깔끔하게 정리된 문서 열람에 적합합니다.

### 섹션 4 — Pydantic 입력 검증

**Q1. `text: str`과 `text: str = "기본값"`의 차이는 무엇입니까?**

- `text: str` → **필수 필드**. 요청에 없으면 422 에러(Field required)가 발생합니다.
- `text: str = "기본값"` → **선택적 필드**. 요청에 없으면 기본값이 사용됩니다.

**Q2. `Field(..., min_length=1, max_length=5000)`에서 `...`은 무엇을 의미합니까?**

`...`(Ellipsis)은 **"이 필드는 필수"**라는 의미입니다. 기본값 자리에 `...`을 넣으면 기본값이 없는 필수 필드가 되어, 누락 시 검증 에러가 발생합니다.

**Q3. 422 에러 응답에서 `loc` 필드는 어떤 정보를 담고 있습니까?**

**에러가 발생한 위치**를 담습니다. 예: `["body", "text"]`는 "요청 본문(body)의 text 필드"에서 검증이 실패했다는 뜻입니다. 클라이언트가 어떤 필드를 고쳐야 하는지 정확히 알 수 있습니다.

**Q4. `response_model`을 지정하면 어떤 이점이 있습니까?**

1. Swagger UI에 응답 스키마가 자동 문서화됩니다.
2. 스키마에 정의되지 않은 필드는 응답에서 자동으로 제거됩니다.
3. 그 결과 내부 데이터(디버그 정보, 민감한 값 등)가 실수로 클라이언트에 노출되는 것을 방지합니다.

### 섹션 5 — 모델 추론 API (Day 2 최종)

**Q1. 모델을 서버 시작 시 한 번만 로드해야 하는 이유는 무엇입니까?**

모델 로드는 파일 I/O가 포함된 무거운 작업이라 수 초가 걸립니다. 요청마다 로드하면 모든 요청의 응답 시간이 수 초로 늘어나므로, 모듈 레벨(서버 시작 시)에서 1회만 로드하고 이후 요청에서는 이미 메모리에 올라온 모델을 재사용해야 합니다.

**Q2. pixel_values가 784개가 아닌 요청이 들어오면 어떤 일이 발생합니까? 이를 처리하는 코드를 직접 작성했습니까?**

Pydantic 스키마에 `min_length=784, max_length=784`를 선언해 두었기 때문에, 개수가 다르면 요청이 추론 코드에 도달하기 전에 **422 에러**(예: "List should have at least 784 items after validation, not 100")로 자동 거부됩니다. 검증 로직 코드는 한 줄도 직접 작성하지 않았습니다 — 스키마 선언만으로 Pydantic이 처리합니다.

**Q3. `HTTPException(status_code=503)`은 어떤 상황에서 사용했습니까? 왜 500이 아니라 503입니까?**

**모델이 로드되지 않은 상태**에서 추론 요청이 들어왔을 때 사용했습니다. 500(Internal Server Error)은 "처리 중 예기치 못한 에러 발생"을 뜻하지만, 503(Service Unavailable)은 "서버는 살아 있으나 현재 서비스를 제공할 수 없는 상태"를 뜻합니다. 모델 미로드는 일시적/준비 안 됨 상태이므로 503이 의미상 정확하고, 클라이언트(또는 로드밸런서)가 "나중에 다시 시도하면 된다"고 판단할 수 있습니다.

**Q4. Swagger UI에서 PredictRequest의 description과 examples가 어디에 표시됩니까?**

`POST /predict`를 펼쳤을 때의 **Request body 스키마 영역**에 표시됩니다. `description`("28x28 이미지의 픽셀 값 (784개). 0.0~1.0 범위.")은 각 필드의 설명으로, `examples`(`[0.0] * 784`)는 [Try it out]을 눌렀을 때 입력란에 미리 채워지는 예시 값으로 나타납니다.
