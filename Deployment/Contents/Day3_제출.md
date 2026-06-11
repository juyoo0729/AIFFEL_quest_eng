# Day 3 — 비동기 처리와 에러 핸들링 제출

> 모델 배포 개론 03 수행 내역 정리
> 제출 항목: ① 섹션 1.5 수행내역 ② 섹션 2, 3 셀 출력 ③ 섹션 5 수행내역

---

## 1. 섹션 1.5 — FastAPI의 요청 처리 방식

FastAPI는 내부적으로 **단일 이벤트 루프(Event Loop)** 에서 동작합니다.

```
[이벤트 루프 — 단일 스레드]

  요청 A 도착 → 핸들러 실행 → ... → 응답 반환
  요청 B 도착 → (A가 끝날 때까지 대기) → 핸들러 실행 → ...
```

FastAPI는 함수의 정의 방식에 따라 다르게 동작합니다:

```python
# 방식 1: 일반 함수 (def)
@app.post("/predict")
def predict(request: PredictRequest):
    result = model(input_tensor)    # 이 동안 이벤트 루프가 블로킹되지 않음
    return result                   # FastAPI가 별도 스레드풀에서 실행해줌

# 방식 2: 비동기 함수 (async def)
@app.post("/predict")
async def predict(request: PredictRequest):
    result = model(input_tensor)    # ⚠️ 이 동안 이벤트 루프가 블로킹됨!
    return result                   # 다른 요청을 받을 수 없음
```

> ⚠️ **핵심 함정:** `async def`로 선언한 함수 안에서 **동기 작업(모델 추론)** 을 수행하면,
> 이벤트 루프가 그 작업이 끝날 때까지 멈추고, 멈춰 있는 동안 다른 요청은 처리되지 않습니다.
> 반면 일반 `def`로 선언하면 FastAPI가 자동으로 별도 스레드에서 실행하지만,
> 스레드 수에 제한이 있어 최적의 해결책은 아닙니다.

---

## 2. 섹션 2 — async/await의 기본 원리 (셀 출력)

### 2.2 실습: 동기 vs 비동기 실행 시간 비교

**동기 방식 (순차 실행)**

```python
def sync_task(name, seconds):
    print(f"  [{name}] 시작")
    time.sleep(seconds)
    print(f"  [{name}] 완료 ({seconds}초)")

sync_task("작업A", 2)
sync_task("작업B", 2)
sync_task("작업C", 2)
```

출력:

```
===== 동기 실행 =====
  [작업A] 시작
  [작업A] 완료 (2초)
  [작업B] 시작
  [작업B] 완료 (2초)
  [작업C] 시작
  [작업C] 완료 (2초)

총 소요 시간: 6.0초
```

**비동기 방식 (동시 실행)**

```python
async def async_task(name, seconds):
    print(f"  [{name}] 시작")
    await asyncio.sleep(seconds)
    print(f"  [{name}] 완료 ({seconds}초)")

await asyncio.gather(
    async_task("작업A", 2),
    async_task("작업B", 2),
    async_task("작업C", 2),
)
```

출력:

```
===== 비동기 실행 =====
  [작업A] 시작
  [작업B] 시작
  [작업C] 시작
  [작업A] 완료 (2초)
  [작업B] 완료 (2초)
  [작업C] 완료 (2초)

총 소요 시간: 2.0초
```

> 결과 비교: 동기 = 2초 + 2초 + 2초 = **6초** (순차), 비동기 = max(2, 2, 2) = **2초** (동시).
> `await`를 만나면 이벤트 루프가 대기를 등록하고 바로 다음 작업을 시작하므로,
> 세 작업의 "시작"이 거의 동시에 출력됩니다.

---

## 3. 섹션 3 — 문제 시연: 동기 추론이 서버를 멈추는 순간 (셀 출력)

### 3.1 실험용 서버 작성

`app/main_sync_problem.py`에 두 가지 엔드포인트를 정의했습니다.
(`/predict/blocking` = `async def` + `time.sleep`, `/predict/threadpool` = 일반 `def`)

```
Writing app/main_sync_problem.py
```

서버 실행:

```
서버 실행됨: http://127.0.0.1:8000
<uvicorn.server.Server at 0x2489b3257d0>
```

### 3.2 실험 1: blocking 엔드포인트에 3개 동시 요청

```
=======================================================
  3개 동시 요청 → http://localhost:8000/predict/blocking
=======================================================
  요청 #1: 11.0초
  요청 #2: 5.0초
  요청 #3: 8.0초

  전체 소요 시간: 11.0초
```

> `async def` 안의 `time.sleep`이 이벤트 루프를 막아, 요청이 순차로 처리됩니다.
> 뒤의 요청일수록 대기 시간이 누적되어 응답이 늦어집니다.

### 3.3 실험 2: threadpool 엔드포인트와 비교

```
=======================================================
  3개 동시 요청 → http://localhost:8000/predict/threadpool
=======================================================
  요청 #1: 5.0초
  요청 #2: 5.0초
  요청 #3: 5.0초

  전체 소요 시간: 5.0초
```

> 일반 `def`는 FastAPI가 별도 스레드풀에서 실행하므로 세 요청이 동시에 처리됩니다.

### 3.4 실험 3: blocking이 헬스체크까지 막는 현상

```
===== /predict/blocking 중 헬스체크 =====
  추론 응답: 5.0초
  헬스체크 응답: 4.5초    ← 단순 상태 확인인데 2.5초 대기!

===== /predict/threadpool 중 헬스체크 =====
  추론 응답: 5.0초
  헬스체크 응답: 2.0초    ← 즉시 응답!
```

> blocking 버전에서는 단순 헬스체크조차 추론이 끝날 때까지 대기합니다.
> 실무에서는 로드밸런서가 헬스체크 실패를 서버 장애로 판단할 수 있어 심각한 문제입니다.

---

## 4. 섹션 5 — 에러 핸들링과 로깅 (수행내역)

### 5.2 글로벌 Exception Handler 작성

`app/error_handlers.py` — 모든 예외를 잡아 안전한 500 응답을 반환하고,
스택 트레이스 등 상세 정보는 서버 로그에만 기록합니다.

```python
def register_error_handlers(app: FastAPI):
    @app.exception_handler(Exception)
    async def general_error_handler(request: Request, exc: Exception):
        logger.error(
            f"에러 발생: {type(exc).__name__}: {exc}\n"
            f"경로: {request.method} {request.url}\n"
            f"스택 트레이스:\n{traceback.format_exc()}"
        )
        return JSONResponse(
            status_code=500,
            content={"success": False, "error": "서버 내부 오류가 발생했습니다."},
        )
```

실행 결과:

```
Writing app/error_handlers.py
```

### 5.3 Python logging 설정

`app/logger_config.py` — `print()` 대신 시간/레벨/모듈명이 포함된 콘솔 로거를 설정합니다.

실행 결과:

```
Writing app/logger_config.py
```

로거 테스트 출력:

```
2026-06-11 12:41:00 INFO     [ml_api] 서버가 시작되었습니다.
2026-06-11 12:41:00 WARNING  [ml_api] GPU 메모리가 80%를 초과했습니다.
2026-06-11 12:41:00 ERROR    [ml_api] 모델 추론 중 에러가 발생했습니다.
```

### 5.4 요청/응답 로깅 미들웨어 작성

`app/middleware.py` — 모든 요청의 메서드, 경로, 응답 시간, 상태 코드를 자동 로깅하며,
상태 코드에 따라 INFO/WARNING/ERROR 레벨을 구분합니다.

```python
class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        duration = round(time.time() - start_time, 3)

        log_message = f"{request.method} {request.url.path} → {response.status_code} ({duration}s)"

        if response.status_code >= 500:
            logger.error(log_message)
        elif response.status_code >= 400:
            logger.warning(log_message)
        else:
            logger.info(log_message)

        response.headers["X-Process-Time"] = str(duration)
        return response
```

실행 결과:

```
Writing app/middleware.py
```

---

수고하셨습니다!
