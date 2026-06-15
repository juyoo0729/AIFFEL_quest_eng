모델 배포 개론 05  
Last modified : 2026.06    
작성 : 박광석 (모두의연구소)  
수정 : 김지성 (모두의연구소)

```python
# 서버 실행 도우미 — 노트북 맨 처음에 한 번 실행하세요.
# 노트북 안에서 uvicorn 서버를 띄우고 멈추는 함수를 정의합니다.
import os, sys, asyncio, threading, time, socket, contextlib
import uvicorn

# 작업 디렉터리를 app/ 가 있는 위치로 맞춥니다 (notebooks/ 안에서 열어도 동작).
if not os.path.isdir('app') and os.path.basename(os.getcwd()) == 'notebooks':
    os.chdir('..')
# 코드를 저장할 폴더를 미리 만들어 둡니다.
for _d in ('app', 'models', 'data', 'frontend'):
    os.makedirs(_d, exist_ok=True)

_SERVERS = {}  # port -> (server, thread)

def _port_open(host, port):
    with contextlib.closing(socket.socket()) as s:
        s.settimeout(0.5)
        return s.connect_ex((host, port)) == 0

def stop_server(port=8000):
    """실행 중인 서버를 멈춥니다."""
    entry = _SERVERS.pop(port, None)
    if not entry:
        return
    server, thread = entry
    server.should_exit = True
    for _ in range(50):
        if not thread.is_alive():
            break
        time.sleep(0.1)

def serve_in_thread(app, host='127.0.0.1', port=8000, log_level='warning'):
    """백그라운드에서 uvicorn 서버를 띄웁니다.

    app: FastAPI 객체 또는 'app.main:app' 같은 import 경로.
    같은 포트에 서버가 이미 있으면 먼저 멈추고 새로 띄웁니다.
    """
    stop_server(port)
    if _port_open(host, port):
        print(f'⚠️ 포트 {port}를 다른 프로세스가 사용 중입니다 (다른 노트북의 서버일 가능성).')
        print('   그 노트북에서 stop_server(8000)을 실행하거나 커널을 종료한 뒤, 이 셀을 다시 실행하세요.')
        return None
    if isinstance(app, str):
        sys.modules.pop(app.split(':')[0], None)   # 파일을 다시 저장한 경우 최신 내용 반영
    for _ in range(50):
        if not _port_open(host, port):
            break
        time.sleep(0.1)
    config = uvicorn.Config(app, host=host, port=port, log_level=log_level, loop='asyncio')
    server = uvicorn.Server(config)
    server.install_signal_handlers = lambda: None
    def _run():
        # Windows는 SelectorEventLoop, 그 외는 기본 이벤트 루프를 사용합니다.
        if sys.platform == 'win32':
            loop = asyncio.SelectorEventLoop()
        else:
            loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        loop.run_until_complete(server.serve())
    thread = threading.Thread(target=_run, daemon=True)
    thread.start()
    _SERVERS[port] = (server, thread)
    for _ in range(40):
        if _port_open(host, port):
            print(f'서버 실행됨: http://{host}:{port}')
            return server
        time.sleep(0.25)
    print('서버가 시작되지 않았습니다. 위 로그를 확인하세요.')
    return server

print('서버 도우미 준비 완료 (serve_in_thread, stop_server)')
```

```
[출력]
서버 도우미 준비 완료 (serve_in_thread, stop_server)

```

# Day 5 — [프로젝트 1] 정형 데이터 예측 서비스

## 1. 프로젝트 개요: 정형 데이터 기반 예측 서비스 설계

---

> **학습 목표**
> - 프로젝트 1의 전체 구조와 최종 결과물을 파악합니다.
> - 캘리포니아 주택 가격 데이터셋을 이해합니다.
> - Day 1~4에서 배운 기술이 프로젝트에서 어떻게 조합되는지 미리 봅니다.

---



### 1.1 오늘 만들 것

Day 1~4까지 MNIST 예제로 조각들을 익혔습니다.
오늘은 **새로운 데이터, 새로운 모델**로 전체 파이프라인을 처음부터 끝까지 직접 구축합니다.

```
[최종 결과물]

브라우저에서:
  1. 주택 정보(면적, 방 수, 위치 등)를 입력합니다.
  2. "가격 예측" 버튼을 누릅니다.
  3. 예상 주택 가격이 표시됩니다.

내부 구조:
  Streamlit (입력 폼) → FastAPI (추론 API) → PyTorch 모델 (가격 예측)
```

---

### 1.2 데이터셋: 캘리포니아 주택 가격

scikit-learn에 내장된 `California Housing` 데이터셋을 사용합니다.
별도 다운로드가 필요 없고, 정형 데이터 예측의 대표적인 예제입니다.

```python
%pip install scikit-learn

```

```python
from sklearn.datasets import fetch_california_housing

data = fetch_california_housing()
print(f"샘플 수: {data.data.shape[0]:,}")
print(f"피처 수: {data.data.shape[1]}")
print(f"타겟: {data.target_names}")
print(f"\n피처 목록:")
for i, name in enumerate(data.feature_names):
    print(f"  {i+1}. {name}")
```

```
[출력]
샘플 수: 20,640
피처 수: 8
타겟: ['MedHouseVal']

피처 목록:
  1. MedInc
  2. HouseAge
  3. AveRooms
  4. AveBedrms
  5. Population
  6. AveOccup
  7. Latitude
  8. Longitude

```

```
샘플 수: 20,640
피처 수: 8
타겟: ['MedHouseVal']

피처 목록:
  1. MedInc      — 중위 소득
  2. HouseAge    — 주택 연식
  3. AveRooms    — 평균 방 수
  4. AveBedrms   — 평균 침실 수
  5. Population  — 인구
  6. AveOccup    — 평균 거주 인원
  7. Latitude    — 위도
  8. Longitude   — 경도
```

- **입력**: 8개 피처 (숫자)
- **출력**: 중위 주택 가격 (단위: $100,000)
- **태스크**: 회귀 (Regression)

> MNIST(Day 1~4)는 이미지 분류였고, 이번 프로젝트는 **정형 데이터 회귀**입니다.
> 데이터 형태가 다르므로 전처리, 스키마, UI가 모두 달라집니다.

---



### 1.3 프로젝트 구조

```
model-serving-course/
├── 📁 app/
│   ├── model_utils.py          ← Day 1~4에서 만든 것 (MNIST용, 오늘은 건드리지 않음)
│   ├── schemas.py              ← Day 2에서 만든 것 (MNIST용, 오늘은 건드리지 않음)
│   ├── ...
│   │
│   ├── housing_model.py        ← 🆕 주택 가격 모델 정의 + 추론 함수
│   ├── housing_schemas.py      ← 🆕 주택 가격 API 스키마
│   └── housing_api.py          ← 🆕 주택 가격 FastAPI 서버
│
├── 📁 frontend/
│   ├── app_dashboard.py        ← Day 4에서 만든 것 (MNIST용)
│   └── app_housing.py          ← 🆕 주택 가격 Streamlit 대시보드
│
├── 📁 models/
│   ├── mnist_state_dict.pth    ← Day 1에서 만든 것
│   ├── housing_model.pth       ← 🆕 주택 가격 모델 가중치
│   └── housing_preprocessing.json ← 🆕 전처리 파라미터 (모델과 함께 배포)
│
└── 📁 notebooks/
    └── day5_project1.ipynb     ← 🆕 오늘의 노트북
```

> 기존 MNIST 파일은 건드리지 않습니다.
> 프로젝트 1의 파일은 모두 `housing_` 접두사를 붙여서 구분합니다.

---



### 1.4 전체 워크플로우

```
[섹션 2] 모델 준비
    │  데이터 로드 → 전처리 → PyTorch 모델 학습 → 저장
    ▼
[섹션 3] FastAPI 백엔드
    │  Pydantic 스키마 → 추론 엔드포인트 → 비동기 처리
    ▼
[섹션 4] Streamlit 프론트엔드
    │  입력 폼 → API 호출 → 결과 시각화
    ▼
[섹션 5] 통합 테스트
    │  정상 요청 / 에러 요청 / 동시 요청 테스트
    ▼
[섹션 6] 회고
```

각 섹션에서 Day 1~4의 어떤 기술이 사용되는지:



![image.png](model05_images/img01.png)

---

### 1.5 시작하기 전에

> ⚠️ **이번 프로젝트의 코드에는 `# *your code*` 주석이 포함되어 있습니다.**
>
> 지금은 전체 코드가 제공되므로 그대로 실행하시면 됩니다.
>
> `# *your code*` 옆의 코드를 직접 작성할 수 있는지 스스로 점검해 보세요.

---

### ✅ 체크포인트

1. 이 프로젝트의 입력과 출력은 각각 무엇입니까?
입력: 주택 정보 — 면적, 방 수, 위치(위도·경도) 등 **여러 개의 숫자 특성(feature)**으로 이루어진 정형 데이터.
출력: 예상 주택 가격 (하나의 연속된 숫자값).

즉 여러 숫자를 넣으면 가격 하나를 예측하는 회귀(regression) 문제입니다. 브라우저에서 주택 정보 입력 → "가격 예측" 버튼 → 예상 가격 표시의 흐름이에요.
2. MNIST 프로젝트와 비교했을 때, 데이터 형태가 어떻게 다릅니까?

| 구분 | MNIST (Day 1~4) | 주택 가격 (Day 5) |
| --- | --- | --- |
| 데이터 종류 | 이미지 (28×28 픽셀) | 정형 데이터 (숫자 특성들) |
| 입력 형태 | 784개 픽셀(2차원 격자) | 면적·방 수·위치 등 소수의 숫자 컬럼 |
| 문제 유형 | 분류 (0~9 중 어느 숫자) | 회귀 (가격이라는 연속값 예측) |
| 출력 | 클래스 라벨 | 연속된 실수값 |

핵심 차이는 이미지 → 정형 데이터, 그리고 분류 → 회귀입니다. 그래서 전처리도 달라져요(MNIST는 픽셀 정규화, 주택은 특성별 mean/std 정규화).
3. 오늘 새로 만들 파일 3개의 이름과 역할을 말할 수 있습니까?
노트북 구조상 app/에 새로 만드는 핵심 파일은 3개입니다:

| 파일 | 역할 |
| --- | --- |
| app/housing_model.py | 🆕 주택 가격 모델 정의 + 추론 함수 |
| app/housing_schemas.py | 🆕 주택 가격 API 스키마 (Pydantic 입력 검증) |
| app/housing_api.py | 🆕 주택 가격 FastAPI 서버 |

여기에 프론트엔드로 frontend/app_housing.py (🆕 Streamlit 대시보드)가 추가됩니다.

---

> **다음 섹션에서는** 캘리포니아 주택 가격 데이터를 로드하고,
> PyTorch 모델을 학습하여 저장합니다.

## 2. 모델 준비: 학습 → 전처리 파이프라인 → 저장

---

> **학습 목표**
> - 캘리포니아 주택 데이터를 로드하고 탐색합니다.
> - 전처리(정규화) 파이프라인을 구성합니다.
> - PyTorch로 회귀 모델을 학습하고 저장합니다.
> - 추론 함수를 모듈로 분리합니다.

---



### 2.1 데이터 로드 및 탐색

```python
import numpy as np
import torch
import torch.nn as nn
from sklearn.datasets import fetch_california_housing
from sklearn.model_selection import train_test_split

# 데이터 로드
data = fetch_california_housing()
X, y = data.data, data.target
feature_names = data.feature_names

print(f"피처 크기: {X.shape}")        # (20640, 8)
print(f"타겟 크기: {y.shape}")        # (20640,)
print(f"타겟 범위: {y.min():.2f} ~ {y.max():.2f} ($100,000 단위)")
print(f"\n피처별 통계:")
for i, name in enumerate(feature_names):
    print(f"  {name:12s}  평균: {X[:, i].mean():10.2f}  표준편차: {X[:, i].std():10.2f}")
```

```
[출력]
피처 크기: (20640, 8)
타겟 크기: (20640,)
타겟 범위: 0.15 ~ 5.00 ($100,000 단위)

피처별 통계:
  MedInc        평균:       3.87  표준편차:       1.90
  HouseAge      평균:      28.64  표준편차:      12.59
  AveRooms      평균:       5.43  표준편차:       2.47
  AveBedrms     평균:       1.10  표준편차:       0.47
  Population    평균:    1425.48  표준편차:    1132.43
  AveOccup      평균:       3.07  표준편차:      10.39
  Latitude      평균:      35.63  표준편차:       2.14
  Longitude     평균:    -119.57  표준편차:       2.00

```

```python
# 학습/테스트 분할
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42  # *your code* — test_size와 random_state 설정
)

print(f"학습 데이터: {X_train.shape[0]:,}개")
print(f"테스트 데이터: {X_test.shape[0]:,}개")
```

```
[출력]
학습 데이터: 16,512개
테스트 데이터: 4,128개

```

---

### 2.2 전처리: 정규화

피처마다 값의 범위가 크게 다릅니다 (소득: 0~15, 인구: 0~35,000).
모델 학습이 안정적으로 수렴하려면 **정규화(Normalization)**가 필요합니다.

```python
# 학습 데이터의 평균/표준편차 계산 (테스트 데이터에는 학습 데이터의 통계를 사용)
train_mean = X_train.mean(axis=0)  # *your code* — 축(axis) 설정
train_std = X_train.std(axis=0)    # *your code* — 축(axis) 설정

print("피처별 평균:", np.round(train_mean, 2))
print("피처별 표준편차:", np.round(train_std, 2))
```

```
[출력]
피처별 평균: [ 3.88000e+00  2.86100e+01  5.44000e+00  1.10000e+00  1.42645e+03
  3.10000e+00  3.56400e+01 -1.19580e+02]
피처별 표준편차: [1.90000e+00 1.26000e+01 2.39000e+00 4.30000e-01 1.13702e+03 1.15800e+01
 2.14000e+00 2.01000e+00]

```

```python
# 정규화 적용
X_train_norm = (X_train - train_mean) / train_std  # *your code* — 정규화 공식
X_test_norm = (X_test - train_mean) / train_std     # 테스트에도 학습 데이터의 통계 사용

print(f"정규화 후 학습 데이터 평균: {X_train_norm.mean(axis=0).round(4)}")  # 거의 0
print(f"정규화 후 학습 데이터 표준편차: {X_train_norm.std(axis=0).round(4)}")  # 거의 1
```

```
[출력]
정규화 후 학습 데이터 평균: [-0. -0.  0. -0. -0. -0.  0. -0.]
정규화 후 학습 데이터 표준편차: [1. 1. 1. 1. 1. 1. 1. 1.]

```

> ⚠️ **왜 학습 데이터의 통계로 테스트 데이터를 정규화합니까?**
>
> 테스트 데이터는 "실제 배포 환경에서 들어올 새로운 데이터"를 시뮬레이션합니다.
> 새로운 데이터의 통계를 미리 알 수 없으므로, 학습 데이터의 통계를 사용합니다.
> 이 평균/표준편차 값은 배포 시에도 함께 저장해야 합니다.

```python
# 텐서 변환
X_train_tensor = torch.FloatTensor(X_train_norm)
y_train_tensor = torch.FloatTensor(y_train).unsqueeze(1)  # *your code* — (N,) → (N,1)
X_test_tensor = torch.FloatTensor(X_test_norm)
y_test_tensor = torch.FloatTensor(y_test).unsqueeze(1)

print(f"X_train 텐서: {X_train_tensor.shape}")  # torch.Size([16512, 8])
print(f"y_train 텐서: {y_train_tensor.shape}")  # torch.Size([16512, 1])
```

```
[출력]
X_train 텐서: torch.Size([16512, 8])
y_train 텐서: torch.Size([16512, 1])

```

---

### 2.3 모델 정의 및 학습

```python
class HousingModel(nn.Module):
    """캘리포니아 주택 가격 예측 모델 (회귀)"""
    def __init__(self, input_dim=8):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, 64),   # *your code* — 입력 차원 → 64
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(64, 32),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(32, 1),           # *your code* — 32 → 1 (회귀 출력)
        )

    def forward(self, x):
        return self.network(x)

model = HousingModel(input_dim=8)
print(f"모델 구조:\n{model}")
print(f"파라미터 수: {sum(p.numel() for p in model.parameters()):,}")
```

```
[출력]
모델 구조:
HousingModel(
  (network): Sequential(
    (0): Linear(in_features=8, out_features=64, bias=True)
    (1): ReLU()
    (2): Dropout(p=0.2, inplace=False)
    (3): Linear(in_features=64, out_features=32, bias=True)
    (4): ReLU()
    (5): Dropout(p=0.2, inplace=False)
    (6): Linear(in_features=32, out_features=1, bias=True)
  )
)
파라미터 수: 2,689

```

```python
# 학습 설정
from torch.utils.data import TensorDataset, DataLoader

train_dataset = TensorDataset(X_train_tensor, y_train_tensor)
train_loader = DataLoader(train_dataset, batch_size=256, shuffle=True)

criterion = nn.MSELoss()                                          # *your code* — 회귀이므로 MSELoss
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)         # *your code* — Adam 옵티마이저

EPOCHS = 50
```

```python
# 학습 루프
model.train()
for epoch in range(1, EPOCHS + 1):
    running_loss = 0.0
    for X_batch, y_batch in train_loader:
        optimizer.zero_grad()
        predictions = model(X_batch)          # *your code* — 순전파
        loss = criterion(predictions, y_batch) # *your code* — 손실 계산
        loss.backward()
        optimizer.step()
        running_loss += loss.item()

    if epoch % 10 == 0:
        avg_loss = running_loss / len(train_loader)
        print(f"Epoch {epoch:3d}/{EPOCHS} — Loss: {avg_loss:.4f}")
```

```
[출력]
Epoch  10/50 — Loss: 0.5402
Epoch  20/50 — Loss: 0.4813
Epoch  30/50 — Loss: 0.4413
Epoch  40/50 — Loss: 0.4156
Epoch  50/50 — Loss: 0.3848

```

```python
# 테스트 평가
model.eval()
with torch.no_grad():
    test_preds = model(X_test_tensor)
    test_loss = criterion(test_preds, y_test_tensor)

    # MAE (Mean Absolute Error) 계산
    mae = torch.abs(test_preds - y_test_tensor).mean().item()

print(f"테스트 MSE:  {test_loss.item():.4f}")
print(f"테스트 MAE:  {mae:.4f} ($100,000 단위)")
print(f"테스트 MAE:  ${mae * 100000:,.0f} (실제 금액)")
```

```
[출력]
테스트 MSE:  0.3263
테스트 MAE:  0.3976 ($100,000 단위)
테스트 MAE:  $39,763 (실제 금액)

```

---

### 2.4 모델 및 전처리 파라미터 저장

```python
import os
os.makedirs("models", exist_ok=True)

# 모델 가중치 저장
torch.save(model.state_dict(), "models/housing_model.pth")  # *your code* — state_dict 저장
print(f"✅ 모델 저장: models/housing_model.pth ({os.path.getsize('models/housing_model.pth')/1024:.1f} KB)")

# 전처리 파라미터 저장 (배포 시 필수!)
preprocessing_params = {
    "mean": train_mean.tolist(),
    "std": train_std.tolist(),
    "feature_names": feature_names,
}

import json
with open("models/housing_preprocessing.json", "w") as f:
    json.dump(preprocessing_params, f, indent=2)

print(f"✅ 전처리 파라미터 저장: models/housing_preprocessing.json")
```

```
[출력]
✅ 모델 저장: models/housing_model.pth (13.3 KB)
✅ 전처리 파라미터 저장: models/housing_preprocessing.json

```

> ⚠️ **전처리 파라미터를 반드시 함께 저장해야 합니다.**
>
> 모델만 저장하고 `train_mean`, `train_std`를 저장하지 않으면,
> 배포 환경에서 정규화를 할 수 없어 잘못된 예측이 나옵니다.
> Day 1에서 ".pth 파일만 전달하면 안 된다"고 배운 것과 같은 맥락입니다.

---



### 2.5 추론 모듈 작성

Day 1에서 `app/model_utils.py`를 만든 것과 동일한 패턴입니다.

```python
%%writefile app/housing_model.py
"""
Day 5 - 주택 가격 예측 모델 정의 + 추론 함수
"""
import json
import torch
import torch.nn as nn
import numpy as np


class HousingModel(nn.Module):
    """캘리포니아 주택 가격 예측 모델"""
    def __init__(self, input_dim=8):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, 64),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(64, 32),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(32, 1),
        )

    def forward(self, x):
        return self.network(x)


class HousingPredictor:
    """모델 로드 + 전처리 + 추론을 캡슐화한 클래스"""

    def __init__(self, model_path: str, preprocessing_path: str):
        # 전처리 파라미터 로드
        with open(preprocessing_path, "r") as f:
            params = json.load(f)
        self.mean = np.array(params["mean"])
        self.std = np.array(params["std"])
        self.feature_names = params["feature_names"]

        # 모델 로드
        self.model = HousingModel(input_dim=len(self.feature_names))
        self.model.load_state_dict(
            torch.load(model_path, map_location="cpu", weights_only=True)
        )
        self.model.eval()

    def predict(self, features: dict) -> dict:
        """
        피처 딕셔너리를 받아 가격을 예측합니다.

        Args:
            features: {"MedInc": 3.5, "HouseAge": 25, ...}
        Returns:
            {"predicted_price": 2.35, "predicted_price_usd": 235000}
        """
        # 피처를 올바른 순서로 배열
        values = [features[name] for name in self.feature_names]  # *your code* — 피처 순서 맞추기

        # 정규화
        values = np.array(values, dtype=np.float32)
        normalized = (values - self.mean) / self.std              # *your code* — 정규화 적용

        # 추론
        input_tensor = torch.FloatTensor(normalized).unsqueeze(0)  # (1, 8)
        with torch.no_grad():
            output = self.model(input_tensor)

        price = output.item()
        price = max(price, 0.0)  # 음수 방지

        return {
            "predicted_price": round(price, 4),
            "predicted_price_usd": int(price * 100000),
        }
```

```
[출력]
Writing app/housing_model.py

```

```python
# 모듈 테스트
import sys
sys.path.insert(0, ".")

from app.housing_model import HousingPredictor

predictor = HousingPredictor(
    model_path="models/housing_model.pth",
    preprocessing_path="models/housing_preprocessing.json",
)

# 테스트 데이터의 첫 번째 샘플로 테스트
sample_features = {name: float(X_test[0, i]) for i, name in enumerate(feature_names)}
print(f"입력 피처: {sample_features}")

result = predictor.predict(sample_features)
print(f"예측 가격: ${result['predicted_price_usd']:,}")
print(f"실제 가격: ${int(y_test[0] * 100000):,}")
```

```
[출력]
입력 피처: {'MedInc': 1.6812, 'HouseAge': 25.0, 'AveRooms': 4.192200557103064, 'AveBedrms': 1.0222841225626742, 'Population': 1392.0, 'AveOccup': 3.8774373259052926, 'Latitude': 36.06, 'Longitude': -119.01}
예측 가격: $70,734
실제 가격: $47,700

```

---

### ✅ 체크포인트

1. 정규화에서 학습 데이터의 통계를 테스트 데이터에도 사용하는 이유는 무엇입니까?
데이터 누수(data leakage)를 막고, 실제 추론 상황과 동일한 조건을 유지하기 위해서입니다.정규화는 (x - mean) / std로 계산하는데, 이때 쓰는 mean·std는 반드시 학습 데이터에서만 구해야 합니다. 이유는:
데이터 누수 방지: 테스트 데이터의 통계까지 정규화에 반영하면, 모델이 평가 시점에 "보면 안 되는 정보"를 미리 엿본 셈이 됩니다. 그러면 성능이 부풀려져서 실제보다 좋아 보입니다.
실전 일관성: 실제 서비스에서 사용자 입력 한 건이 들어올 땐 그 한 건으로 통계를 낼 수 없습니다. 미리 정해둔 학습 데이터의 mean·std로 변환해야, 학습 때와 똑같은 기준으로 모델이 동작합니다.
즉 테스트·실전 입력은 학습 때 정한 고정된 통계로 변환해야 평가가 공정하고 동작이 일관됩니다.
2. 모델 가중치 외에 **함께 저장해야 하는 것**은 무엇이고, 왜 필요합니까?
전처리 파라미터(mean, std)와 피처 이름·순서 정보를 함께 저장해야 합니다.
함께 저장할 것이유mean, std (정규화 통계)추론 시 입력을 학습 때와 똑같이 정규화해야 함. 이 값이 없으면 같은 입력도 다른 숫자로 변환되어 예측이 엉망이 됨feature_names (피처 순서)입력 특성을 모델이 학습한 순서대로 배열하기 위함
핵심 이유: 모델 가중치는 "정규화된 입력"을 전제로 학습됩니다. 추론할 때 가중치만 있고 mean/std가 없으면, 원본 입력을 그대로 넣게 되어 학습 때와 완전히 다른 스케일이 되고 예측이 무의미해집니다. 그래서 모델과 전처리 파라미터는 한 세트로 묶어 저장해야 합니다.
3. `HousingPredictor.predict()`에서 피처를 `self.feature_names` 순서로 배열하는 이유는?
모델은 피처의 "순서"로 학습했기 때문에, 추론 때도 정확히 같은 순서로 넣어야 하기 때문입니다.
신경망은 입력 벡터의 **위치(인덱스)**로 각 특성을 구분합니다. 예를 들어 0번 자리가 MedInc(소득), 1번 자리가 HouseAge(주택 연식)로 학습했다면, 가중치도 그 순서에 맞춰 만들어졌습니다.
만약 추론 때 사용자 입력을 다른 순서로(예: 연식을 0번에, 소득을 1번에) 넣으면:

모델은 연식을 소득으로 착각하고 계산합니다.
입력 형태(shape)는 맞으니 에러도 안 나고 조용히 엉뚱한 예측을 내놓습니다 — 가장 잡기 어려운 버그.

그래서 self.feature_names라는 고정된 순서 기준에 맞춰 입력을 재배열하면, 사용자가 어떤 순서로 보내든(JSON은 순서가 보장 안 됨) 항상 모델이 기대하는 순서로 정렬되어 안전합니다.

---

> **다음 섹션에서는** 이 `HousingPredictor`를 FastAPI 엔드포인트에 연결합니다.


## 3. FastAPI 백엔드: 추론 엔드포인트 + Pydantic 스키마

---

> **학습 목표**
> - 주택 가격 예측용 Pydantic 스키마를 설계합니다.
> - FastAPI 엔드포인트를 구현하고 비동기 처리를 적용합니다.
> - Swagger UI에서 API를 테스트합니다.

---


### 3.1 Pydantic 스키마 설계

```python
%%writefile app/housing_schemas.py
"""
Day 5 - 주택 가격 예측 API 스키마
"""
from pydantic import BaseModel, Field


class HousingRequest(BaseModel):
    """주택 가격 예측 요청"""
    MedInc: float = Field(..., gt=0, description="중위 소득")                        # *your code* — gt=0 설정
    HouseAge: float = Field(..., ge=0, le=100, description="주택 연식 (년)")          # *your code* — ge, le 범위
    AveRooms: float = Field(..., gt=0, description="평균 방 수")
    AveBedrms: float = Field(..., gt=0, description="평균 침실 수")
    Population: float = Field(..., gt=0, description="인구")
    AveOccup: float = Field(..., gt=0, description="평균 거주 인원")
    Latitude: float = Field(..., ge=32, le=42, description="위도 (캘리포니아 범위)")   # *your code* — 캘리포니아 위도 범위
    Longitude: float = Field(..., ge=-125, le=-114, description="경도 (캘리포니아 범위)")

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "MedInc": 3.5,
                    "HouseAge": 25.0,
                    "AveRooms": 5.0,
                    "AveBedrms": 1.0,
                    "Population": 1500.0,
                    "AveOccup": 3.0,
                    "Latitude": 37.5,
                    "Longitude": -122.0,
                }
            ]
        }
    }


class HousingResponse(BaseModel):
    """주택 가격 예측 응답"""
    success: bool = Field(description="요청 처리 성공 여부")
    predicted_price: float = Field(description="예측 가격 ($100,000 단위)")
    predicted_price_usd: int = Field(description="예측 가격 (USD)")
    input_features: dict = Field(description="입력된 피처 값")
```

```
[출력]
Overwriting app/housing_schemas.py

```

> Day 2에서 배운 `Field()`를 그대로 활용합니다.  
> 캘리포니아 데이터이므로 위도/경도에 범위 제한을 넣어 잘못된 입력을 방지합니다.

---



### 3.2 FastAPI 서버 구현

```python
%%writefile app/housing_api.py
"""
Day 5 - 주택 가격 예측 FastAPI 서버
"""
import asyncio
from contextlib import asynccontextmanager
from concurrent.futures import ThreadPoolExecutor

from fastapi import FastAPI, HTTPException

from app.housing_schemas import HousingRequest, HousingResponse
from app.housing_model import HousingPredictor
from app.logger_config import setup_logger
from app.error_handlers import register_error_handlers
from app.middleware import RequestLoggingMiddleware


# ===== 설정 =====
logger = setup_logger("housing_api")

# 추론 전용 스레드풀 (Day 3에서 배운 패턴)
inference_executor = ThreadPoolExecutor(max_workers=4, thread_name_prefix="housing")  # *your code* — 스레드풀 생성

# ===== 모델 로드 =====
MODEL_PATH = "models/housing_model.pth"
PREPROCESS_PATH = "models/housing_preprocessing.json"
predictor = None


# ===== 수명 주기(lifespan): 시작 시 모델 로드 (구 @app.on_event 대체) =====
@asynccontextmanager
async def lifespan(app: FastAPI):
    global predictor
    logger.info("주택 가격 모델 로드 중...")
    predictor = HousingPredictor(MODEL_PATH, PREPROCESS_PATH)  # *your code* — HousingPredictor 인스턴스 생성
    logger.info("모델 로드 완료")
    yield
    # (서버 종료 시 정리할 자원이 있으면 yield 뒤에 작성)


app = FastAPI(
    title="California Housing Price API",
    description="캘리포니아 주택 가격을 예측하는 API",
    version="1.0.0",
    lifespan=lifespan,
)

app.add_middleware(RequestLoggingMiddleware)
register_error_handlers(app)


# ===== 엔드포인트 =====

@app.get("/health", tags=["System"])
async def health_check():
    return {
        "status": "healthy" if predictor is not None else "loading",
        "model": "California Housing",
    }


@app.post("/predict", response_model=HousingResponse, tags=["Prediction"])
async def predict_housing(request: HousingRequest):
    """주택 정보를 받아 가격을 예측합니다."""
    if predictor is None:
        raise HTTPException(status_code=503, detail="모델이 아직 로드되지 않았습니다.")

    # 요청 데이터를 딕셔너리로 변환
    features = request.model_dump()  # *your code* — Pydantic 모델 → dict

    try:
        # 추론 (별도 스레드에서 실행 — Day 3 패턴)
        loop = asyncio.get_running_loop()
        result = await loop.run_in_executor(       # *your code* — run_in_executor 사용
            inference_executor,
            predictor.predict,
            features,
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"추론 실패: {str(e)}")

    return HousingResponse(
        success=True,
        predicted_price=result["predicted_price"],
        predicted_price_usd=result["predicted_price_usd"],
        input_features=features,
    )

```

```
[출력]
Writing app/housing_api.py

```

---

### 3.3 서버 실행 및 Swagger 테스트

```python
%pip install uvicorn

```

```
[출력]
Requirement already satisfied: uvicorn in c:\ai_study\model-serving-course\.venv\lib\site-packages (0.30.0)
Requirement already satisfied: click>=7.0 in c:\ai_study\model-serving-course\.venv\lib\site-packages (from uvicorn) (8.4.1)
Requirement already satisfied: h11>=0.8 in c:\ai_study\model-serving-course\.venv\lib\site-packages (from uvicorn) (0.16.0)
Requirement already satisfied: colorama in c:\ai_study\model-serving-course\.venv\lib\site-packages (from click>=7.0->uvicorn) (0.4.6)
Note: you may need to restart the kernel to use updated packages.

```

```python
# 서버 실행 (같은 포트에 서버가 떠 있으면 자동으로 멈추고 새로 띄웁니다)
serve_in_thread("app.housing_api:app", port=8000)
```

```
[출력]
2026-06-15 12:30:19 INFO     [housing_api] 주택 가격 모델 로드 중...
2026-06-15 12:30:19 INFO     [housing_api] 모델 로드 완료
서버 실행됨: http://127.0.0.1:8000

<uvicorn.server.Server at 0x1a1d760ee90>
```

```python
import requests, json

# 헬스체크
resp = requests.get("http://localhost:8000/health")
print(f"헬스체크: {resp.json()}")
```

```
[출력]
헬스체크: {'status': 'healthy', 'model': 'California Housing'}

```

```python
# 추론 테스트
sample_request = {
    "MedInc": 3.5,
    "HouseAge": 25.0,
    "AveRooms": 5.0,
    "AveBedrms": 1.0,
    "Population": 1500.0,
    "AveOccup": 3.0,
    "Latitude": 37.5,
    "Longitude": -122.0,
}

resp = requests.post("http://localhost:8000/predict", json=sample_request)
result = resp.json()

print(f"상태 코드: {resp.status_code}")
print(f"예측 가격: ${result['predicted_price_usd']:,}")
print(f"전체 응답:")
print(json.dumps(result, indent=2, ensure_ascii=False))
```

```
[출력]
상태 코드: 200
예측 가격: $183,526
전체 응답:
{
  "success": true,
  "predicted_price": 1.8353,
  "predicted_price_usd": 183526,
  "input_features": {
    "MedInc": 3.5,
    "HouseAge": 25.0,
    "AveRooms": 5.0,
    "AveBedrms": 1.0,
    "Population": 1500.0,
    "AveOccup": 3.0,
    "Latitude": 37.5,
    "Longitude": -122.0
  }
}

```

```python
# 에러 테스트: 필수 필드 누락
resp = requests.post("http://localhost:8000/predict", json={"MedInc": 3.5})
print(f"필드 누락 → 상태: {resp.status_code}")

# 에러 테스트: 범위 초과
resp = requests.post("http://localhost:8000/predict", json={
    **sample_request, "Latitude": 50.0  # 캘리포니아 범위 초과
})
print(f"범위 초과 → 상태: {resp.status_code}")
```

```
[출력]
POST /predict -> 422 (0.001s)

필드 누락 → 상태: 422

POST /predict -> 422 (0.0s)

범위 초과 → 상태: 422

```

---

### ✅ 체크포인트

1. `HousingRequest`에서 `Latitude`에 `ge=32, le=42` 제한을 넣은 이유는?
캘리포니아의 실제 위도 범위를 벗어난 입력을 문 앞에서 막기 위해서입니다.이 데이터셋은 캘리포니아 주택 가격이고, 캘리포니아의 위도는 대략 32°~42°N 사이에 있습니다. ge=32, le=42(32 이상, 42 이하)는 이 지리적 범위를 벗어난 값을 검증 단계에서 거르는 제약입니다.
왜 필요한가:모델은 캘리포니아 범위의 데이터로만 학습했습니다. 위도 100 같은 말도 안 되는 값이 들어오면 학습 분포 밖이라 예측이 무의미합니다.
잘못된 입력은 추론까지 가기 전에 422 에러로 즉시 거부하는 게 안전합니다. (Day 2의 "잘못된 입력은 문 앞에서 막는다" 원칙)
즉, 도메인 지식(캘리포니아 위경도)을 Pydantic 검증 규칙으로 옮겨 담은 거예요.
2. `request.model_dump()`는 어떤 역할을 합니까?
Pydantic 모델 객체를 일반 파이썬 딕셔너리(dict)로 변환합니다.
python# HousingRequest 객체 → dict
data = request.model_dump()
# {"MedInc": 8.3, "HouseAge": 41, "Latitude": 37.8, ...}
API로 들어온 요청은 HousingRequest라는 Pydantic 객체 형태인데, 이걸 추론 함수에 넘기거나 전처리하려면 보통 dict나 배열로 다뤄야 편합니다. model_dump()가 그 변환을 해줍니다.
참고: Pydantic v1의 .dict()가 v2에서 **.model_dump()**로 이름이 바뀐 것입니다. 같은 역할이에요.
이렇게 dict로 만든 뒤 feature_names 순서대로 값을 뽑아 모델 입력 벡터를 구성합니다.

3. `run_in_executor`를 사용하지 않으면 어떤 문제가 발생할 수 있습니까?
무거운 추론이 이벤트 루프를 직접 붙들어, 서버 전체가 멈추는 문제가 생깁니다.
async def 핸들러 안에서 run_in_executor 없이 추론을 직접 실행하면, 그 추론이 루프 위에서 돌면서 일꾼(이벤트 루프)을 점유합니다. 일꾼은 한 명뿐이라 그동안:

다른 요청이 전부 대기 — 동시 요청이 순차 처리되어, 3명이 오면 9초가 걸리는 식.
헬스체크(/health)까지 막힘 — 로드밸런서가 무응답을 "죽었다"로 판단해 멀쩡한 서버를 트래픽에서 빼버리고, 도미노 장애로 번질 수 있음.

run_in_executor는 추론을 옆방(스레드풀)으로 떼어내 루프를 비워두기 때문에, 추론이 도는 중에도 새 요청·헬스체크가 즉시 처리됩니다.

---

> **다음 섹션에서는** Streamlit 프론트엔드를 만들어 이 API와 연결합니다.


## 4. Streamlit 프론트엔드: 입력 폼 → API 호출 → 결과 시각화

---

> **학습 목표**
> - 정형 데이터 입력 폼을 Streamlit으로 구현합니다.
> - Day 4에서 배운 패턴으로 FastAPI를 호출합니다.
> - 예측 결과를 시각적으로 표시합니다.

---



### 4.1 대시보드 코드 작성

```python
%%writefile frontend/app_housing.py
"""
Day 5 - 캘리포니아 주택 가격 예측 대시보드
"""
import streamlit as st
import requests

# ===== 페이지 설정 =====
st.set_page_config(
    page_title="주택 가격 예측",
    page_icon="🏠",
    layout="wide",
)

# ===== API 호출 함수 (Day 4 패턴) =====
API_BASE = "http://localhost:8000"

def call_api(url, json_data=None, method="post"):
    try:
        if method == "get":
            resp = requests.get(url, timeout=10)
        else:
            resp = requests.post(url, json=json_data, timeout=30)
        resp.raise_for_status()                      # *your code* — HTTP 에러 시 예외 발생
        return resp.json()
    except requests.exceptions.ConnectionError:
        st.error("🔌 **서버에 연결할 수 없습니다.** FastAPI 서버를 실행하세요.")
        return None
    except requests.exceptions.HTTPError as e:
        st.error(f"❌ **서버 에러** (HTTP {e.response.status_code})")
        return None
    except Exception as e:
        st.error(f"❌ **오류:** {type(e).__name__}")
        return None


# ===== 사이드바 =====
with st.sidebar:
    st.header("⚙️ 설정")

    health = call_api(f"{API_BASE}/health", method="get")
    if health and health.get("status") == "healthy":
        st.success("🟢 서버 연결됨")
        server_ok = True
    else:
        st.error("🔴 서버 연결 실패")
        server_ok = False

    st.divider()
    st.caption("California Housing Price Predictor")
    st.caption("Day 5 — 프로젝트 1")


# ===== 메인 영역 =====
st.title("🏠 캘리포니아 주택 가격 예측")
st.write("주택 정보를 입력하면 예상 가격을 예측합니다.")

col_input, col_result = st.columns(2)

# ----- 입력 영역 -----
with col_input:
    st.subheader("📋 주택 정보 입력")

    # 소득 & 주택 연식
    c1, c2 = st.columns(2)
    with c1:
        med_inc = st.number_input(
            "중위 소득 (MedInc)",
            min_value=0.1, max_value=20.0, value=3.5, step=0.1,   # *your code* — 범위와 기본값
        )
    with c2:
        house_age = st.number_input(
            "주택 연식 (HouseAge)",
            min_value=0.0, max_value=100.0, value=25.0, step=1.0,
        )

    # 방 수 & 침실 수
    c1, c2 = st.columns(2)
    with c1:
        ave_rooms = st.number_input(
            "평균 방 수 (AveRooms)",
            min_value=0.1, max_value=50.0, value=5.0, step=0.1,
        )
    with c2:
        ave_bedrms = st.number_input(
            "평균 침실 수 (AveBedrms)",
            min_value=0.1, max_value=20.0, value=1.0, step=0.1,
        )

    # 인구 & 거주 인원
    c1, c2 = st.columns(2)
    with c1:
        population = st.number_input(
            "인구 (Population)",
            min_value=1.0, max_value=50000.0, value=1500.0, step=100.0,
        )
    with c2:
        ave_occup = st.number_input(
            "평균 거주 인원 (AveOccup)",
            min_value=0.1, max_value=20.0, value=3.0, step=0.1,
        )

    # 위치
    c1, c2 = st.columns(2)
    with c1:
        latitude = st.number_input(
            "위도 (Latitude)",
            min_value=32.0, max_value=42.0, value=37.5, step=0.1,   # *your code* — 캘리포니아 범위
        )
    with c2:
        longitude = st.number_input(
            "경도 (Longitude)",
            min_value=-125.0, max_value=-114.0, value=-122.0, step=0.1,
        )


# ----- 결과 영역 -----
with col_result:
    st.subheader("📊 예측 결과")

    if not server_ok:
        st.error("서버에 연결할 수 없습니다.")
    else:
        if st.button("🚀 가격 예측", type="primary", use_container_width=True):
            # 요청 데이터 구성
            request_data = {                            # *your code* — 입력값을 dict로 구성
                "MedInc": med_inc,
                "HouseAge": house_age,
                "AveRooms": ave_rooms,
                "AveBedrms": ave_bedrms,
                "Population": population,
                "AveOccup": ave_occup,
                "Latitude": latitude,
                "Longitude": longitude,
            }

            with st.spinner("예측 중..."):
                result = call_api(f"{API_BASE}/predict", json_data=request_data)

            if result:
                st.session_state["last_housing_result"] = result

        # 결과 표시
        if "last_housing_result" in st.session_state:
            result = st.session_state["last_housing_result"]

            # 가격 메트릭
            st.metric(
                label="예상 주택 가격",
                value=f"${result['predicted_price_usd']:,}",
            )

            st.caption(f"모델 출력값: {result['predicted_price']} ($100,000 단위)")

            # 입력 피처 확인
            with st.expander("📋 입력된 피처 확인"):
                for key, value in result["input_features"].items():
                    st.write(f"**{key}**: {value}")
```

```
[출력]
Writing frontend/app_housing.py

GET / -> 404 (0.0s)
GET /favicon.ico -> 404 (0.001s)

```

---

### 4.2 실행 및 테스트

아래는 터미널에서 실행하는 명령입니다.

```bash
# 터미널 1: 백엔드
uvicorn app.housing_api:app --port 8000

# 터미널 2: 프론트엔드
streamlit run frontend/app_housing.py --server.port 8501
```

```python
import sys, subprocess, time, socket, contextlib, tempfile, os, signal

def _pids_on_port(port):
    """포트를 LISTEN 중인 프로세스 PID 목록. (Windows/macOS/Linux 공통)"""
    pids = set()
    try:
        if os.name == "nt":                  # Windows: netstat
            out = subprocess.run(["netstat", "-ano"], capture_output=True, text=True).stdout
            for ln in out.splitlines():
                p = ln.split()
                if len(p) >= 5 and p[0] == "TCP" and p[1].endswith(f":{port}") and p[3] == "LISTENING":
                    pids.add(int(p[-1]))
        else:                                # macOS / Linux: lsof
            out = subprocess.run(["lsof", "-ti", f"tcp:{port}", "-sTCP:LISTEN"],
                                 capture_output=True, text=True).stdout
            pids.update(int(x) for x in out.split())
    except (FileNotFoundError, ValueError):
        pass
    return pids

def _free_port(port):
    """포트를 점유한 (이전) 서버를 종료하고, 포트가 닫힐 때까지 기다린다."""
    pids = _pids_on_port(port)
    if not pids:
        return
    print(f"🔄 포트 {port}의 기존 서버 종료: PID {sorted(pids)}")

    def kill(pid, hard=False):
        try:
            if os.name == "nt":
                subprocess.run(["taskkill", "/F", "/PID", str(pid)], capture_output=True)
            else:
                os.kill(pid, signal.SIGKILL if hard else signal.SIGTERM)
        except (ProcessLookupError, PermissionError):
            pass

    for pid in pids:
        kill(pid)
    for _ in range(20):                      # 최대 5초 동안 닫히길 기다림
        if not _pids_on_port(port):
            time.sleep(0.3)                  # 소켓 정리(TIME_WAIT) 여유
            return
        time.sleep(0.25)
    for pid in _pids_on_port(port):          # 그래도 안 닫히면 강제 종료
        kill(pid, hard=True)
    time.sleep(0.5)

def run_streamlit(script, port=8501):
    """Streamlit을 백그라운드로 띄우고 '실제로 떴는지'까지 확인한다. (Windows/macOS/Linux 공통)"""
    def port_open(p):
        with contextlib.closing(socket.socket()) as s:
            s.settimeout(0.5)
            return s.connect_ex(("127.0.0.1", p)) == 0

    _free_port(port)                         # 이미 떠 있으면 닫고 새로 띄운다 — 코드 변경이 바로 반영되도록

    # 로그는 파일로 — 파이프가 가득 차 서버가 멈추는 일을 막고, 실패 시 원인을 읽을 수 있다
    log_path = os.path.join(tempfile.gettempdir(), f"streamlit_{port}.log")
    log = open(log_path, "w", encoding="utf-8")

    proc = subprocess.Popen(
        [sys.executable, "-m", "streamlit", "run", script,
         "--server.port", str(port),
         "--server.headless", "true"],       # 최초 실행 '이메일 프롬프트'를 건너뜀
        stdout=log, stderr=subprocess.STDOUT, #   (없으면 입력을 기다리다 조용히 죽음)
    )
    for _ in range(60):                      # 최대 15초, 0.25초 간격 확인
        if proc.poll() is not None:          # 일찍 죽음 → 로그를 보여줌
            log.close()
            print(f"❌ Streamlit이 종료됨 (code {proc.returncode}) — 로그:")
            print(open(log_path, encoding="utf-8").read()[-2000:])
            return proc
        if port_open(port):
            print(f"✅ 프론트엔드: http://localhost:{port}")
            print(f"   (로그: {log_path})")
            return proc
        time.sleep(0.25)
    proc.terminate(); log.close()
    print(f"❌ 15초 내에 포트가 열리지 않음 — 로그:")
    print(open(log_path, encoding="utf-8").read()[-2000:])
    return proc

proc = run_streamlit("frontend/app_housing.py", port=8501)

# Colab: from google.colab import output
# output.serve_kernel_port_as_iframe(8501)
```

```
[출력]
✅ 프론트엔드: http://localhost:8501
   (로그: C:\Users\yoehe\AppData\Local\Temp\streamlit_8501.log)

```

![image.png](model05_images/img02.png)

#### 테스트 시나리오

```
[테스트 1] 기본 흐름
  1. 사이드바 서버 상태 🟢 확인
  2. 기본값 그대로 "🚀 가격 예측" 클릭
  3. 예상 가격이 표시되는지 확인

[테스트 2] 값 변경
  1. 중위 소득을 8.0으로 올리기 → 가격 예측 → 가격 상승 확인
  2. 중위 소득을 1.0으로 내리기 → 가격 예측 → 가격 하락 확인

[테스트 3] 에러 상황
  1. 백엔드 서버 종료 → 사이드바 🔴 확인
```

---

### ✅ 체크포인트

1. MNIST 대시보드(Day 4)와 비교했을 때 입력 방식이 어떻게 다릅니까?
MNIST는 이미지 업로드, 주택 가격은 숫자 입력 폼입니다.

| 구분 | MNIST 대시보드 (Day 4) | 주택 가격 대시보드 (Day 5) |
| --- | --- | --- |
| 입력 방식 | st.file_uploader()로 이미지 파일 업로드 | st.number_input() 여러 개로 숫자 직접 입력 |
| 입력 데이터 | 28×28 픽셀 이미지 한 장 | 면적·방 수·위치 등 여러 개의 숫자 특성 |
| UI 형태 | 파일 업로드 버튼 | 특성별 입력 칸이 나열된 입력 폼 |

이미지 한 덩어리를 통째로 보내던 것에서, 여러 개의 숫자 필드를 폼으로 받아 묶어 보내는 방식으로 바뀐 거예요. 정형 데이터의 특성을 반영한 차이입니다.
2. `st.number_input()`에서 `min_value`, `max_value`를 설정하는 이유는?
잘못된 범위의 입력을 UI 단계에서 미리 막기 위해서입니다.
min_value, max_value를 설정하면 사용자가 그 범위를 벗어난 값을 아예 입력할 수 없게 됩니다(또는 자동으로 범위 안으로 제한됨). 이유는:
1차 방어선: 위도 100 같은 말도 안 되는 값을 서버로 보내기 전에 브라우저에서 즉시 차단. 서버 검증(Pydantic ge/le)이 2차 방어선이라면, 이건 사용자에게 더 빠르고 친절한 1차 방어선입니다.
사용자 경험: 422 에러를 받고 나서 고치는 것보다, 입력 단계에서 가능한 범위를 알려주는 게 훨씬 낫습니다.
핵심: UI 검증(Streamlit) + 서버 검증(Pydantic)을 이중으로 두는 것. UI 검증은 우회될 수 있으니(직접 API 호출 등), 서버 검증은 여전히 필수입니다. UI는 편의, 서버는 보안.
3. `request_data` 딕셔너리의 키 이름이 `HousingRequest` 스키마의 필드 이름과 정확히 일치해야 하는 이유는?
Pydantic이 키 이름으로 각 필드를 매칭해 검증하기 때문입니다.
Streamlit에서 보내는 JSON은 이렇게 생겼습니다:
pythonrequest_data = {
    "MedInc": 8.3,
    "HouseAge": 41,
    "Latitude": 37.8,
    # ...
}
FastAPI가 이걸 받으면 HousingRequest 스키마의 필드 이름과 키를 하나씩 대조해서 값을 채웁니다. 만약 키 이름이 다르면(예: median_income vs MedInc):

Pydantic이 해당 필드를 못 찾아 검증 실패 → 422 에러.
혹은 필수 필드 누락으로 요청 자체가 거부됨.

JSON은 순서가 아니라 키(이름)로 값을 찾기 때문에, 보내는 쪽(Streamlit request_data)과 받는 쪽(HousingRequest 필드)의 이름이 글자 하나까지 정확히 일치해야 합니다. 대소문자도 구분되므로 Latitude와 latitude는 다른 키로 취급됩니다.
---

> **다음 섹션에서는** 전체 서비스를 통합 테스트합니다.

## 5. 통합 테스트 및 디버깅

---

> **학습 목표**
> - 전체 서비스(모델 → API → UI)가 정상 동작하는지 체계적으로 테스트합니다.
> - 다양한 입력으로 모델의 동작을 확인합니다.
> - 에러 상황에서 서버가 안정적으로 동작하는지 검증합니다.

---



### 5.1 API 레벨 통합 테스트

```python
# ⚠️ FastAPI 서버가 실행 중이어야 합니다.

import requests
import json
import time

API_BASE = "http://localhost:8000"

print("=" * 60)
print("  통합 테스트")
print("=" * 60)
```

```
[출력]
============================================================
  통합 테스트
============================================================

```

#### 테스트 1: 정상 요청

```python
# 다양한 입력으로 모델 동작 확인
test_cases = [
    {"name": "저소득 지역", "MedInc": 1.5, "HouseAge": 40, "AveRooms": 4.0, "AveBedrms": 1.0,
     "Population": 2000, "AveOccup": 3.5, "Latitude": 34.0, "Longitude": -118.0},
    {"name": "고소득 지역", "MedInc": 10.0, "HouseAge": 10, "AveRooms": 8.0, "AveBedrms": 2.0,
     "Population": 500, "AveOccup": 2.0, "Latitude": 37.8, "Longitude": -122.4},
    {"name": "평균적 주택", "MedInc": 3.5, "HouseAge": 25, "AveRooms": 5.0, "AveBedrms": 1.0,
     "Population": 1500, "AveOccup": 3.0, "Latitude": 37.5, "Longitude": -122.0},
]

print("\n[테스트 1] 정상 요청 — 다양한 입력")
print(f"{'케이스':<15} {'예측 가격':>12}")
print("-" * 30)

for case in test_cases:
    name = case.pop("name")
    resp = requests.post(f"{API_BASE}/predict", json=case)    # *your code* — POST 요청
    result = resp.json()
    print(f"{name:<15} ${result['predicted_price_usd']:>10,}")
    case["name"] = name  # 복원
```

```
[출력]

[테스트 1] 정상 요청 — 다양한 입력
케이스                    예측 가격
------------------------------
저소득 지역          $   104,385
고소득 지역          $   479,974
평균적 주택          $   183,526

```

#### 테스트 2: 에러 상황

```python
print("\n[테스트 2] 에러 상황")

# 필수 필드 누락
resp = requests.post(f"{API_BASE}/predict", json={"MedInc": 3.5})
print(f"  필드 누락      → HTTP {resp.status_code}")

# 범위 초과 (위도)
bad_request = {
    "MedInc": 3.5, "HouseAge": 25, "AveRooms": 5, "AveBedrms": 1,
    "Population": 1500, "AveOccup": 3, "Latitude": 50.0, "Longitude": -122.0,  # 위도 초과
}
resp = requests.post(f"{API_BASE}/predict", json=bad_request)
print(f"  위도 범위 초과  → HTTP {resp.status_code}")

# 음수 값
bad_request2 = {
    "MedInc": -1.0, "HouseAge": 25, "AveRooms": 5, "AveBedrms": 1,
    "Population": 1500, "AveOccup": 3, "Latitude": 37.5, "Longitude": -122.0,
}
resp = requests.post(f"{API_BASE}/predict", json=bad_request2)
print(f"  소득 음수      → HTTP {resp.status_code}")

# JSON이 아닌 요청
resp = requests.post(f"{API_BASE}/predict", data="not json")
print(f"  잘못된 포맷    → HTTP {resp.status_code}")
```

```
[출력]

[테스트 2] 에러 상황

POST /predict -> 422 (0.0s)

  필드 누락      → HTTP 422

POST /predict -> 422 (0.001s)

  위도 범위 초과  → HTTP 422

POST /predict -> 422 (0.0s)

  소득 음수      → HTTP 422

POST /predict -> 422 (0.001s)

  잘못된 포맷    → HTTP 422

```

#### 테스트 3: 동시 요청

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def send_predict(i):
    case = test_cases[i % len(test_cases)].copy()
    case.pop("name", None)
    start = time.time()
    resp = requests.post(f"{API_BASE}/predict", json=case, timeout=30)
    return {"id": i+1, "elapsed": round(time.time() - start, 3), "status": resp.status_code}

print("\n[테스트 3] 동시 요청 (8개)")
start = time.time()
with ThreadPoolExecutor(max_workers=8) as ex:
    futures = [ex.submit(send_predict, i) for i in range(8)]
    results = [f.result() for f in as_completed(futures)]

total = round(time.time() - start, 2)
for r in sorted(results, key=lambda x: x["id"]):
    print(f"  요청 #{r['id']}: {r['elapsed']}초 (HTTP {r['status']})")
print(f"  전체: {total}초")
```

```
[출력]

[테스트 3] 동시 요청 (8개)
  요청 #1: 2.029초 (HTTP 200)
  요청 #2: 2.025초 (HTTP 200)
  요청 #3: 2.043초 (HTTP 200)
  요청 #4: 2.04초 (HTTP 200)
  요청 #5: 2.039초 (HTTP 200)
  요청 #6: 2.039초 (HTTP 200)
  요청 #7: 2.049초 (HTTP 200)
  요청 #8: 2.02초 (HTTP 200)
  전체: 2.06초

```

#### 테스트 4: 헬스체크

```python
print("\n[테스트 4] 헬스체크")
resp = requests.get(f"{API_BASE}/health")
print(f"  상태: {resp.json()}")
```

```
[출력]

[테스트 4] 헬스체크
  상태: {'status': 'healthy', 'model': 'California Housing'}

```

---

### 5.2 테스트 결과 종합

```python
print("\n" + "=" * 60)
print("  테스트 결과 종합")
print("=" * 60)
print("  ✅ 정상 요청: 다양한 입력에서 합리적인 가격 반환")
print("  ✅ 에러 처리: 잘못된 입력에 422/400 반환, 서버 안 죽음")
print("  ✅ 동시 처리: 8개 동시 요청 정상 처리")
print("  ✅ 헬스체크: 서버 상태 정상")
```

```
[출력]

============================================================
  테스트 결과 종합
============================================================
  ✅ 정상 요청: 다양한 입력에서 합리적인 가격 반환
  ✅ 에러 처리: 잘못된 입력에 422/400 반환, 서버 안 죽음
  ✅ 동시 처리: 8개 동시 요청 정상 처리
  ✅ 헬스체크: 서버 상태 정상

```

---

## 6. 회고: 무엇이 어려웠고, 어떻게 개선할 수 있는가?

---

### 6.1 오늘 만든 것 정리

![image.png](model05_images/img03.png)



### 6.2 Day 1~4와의 연결



![image.png](model05_images/img04.png)

### 6.3 개선할 수 있는 점


![image.png](model05_images/img05.png)



> 이 개선점들은 이 과정의 나머지 Day에서 하나씩 다루거나,
> 이후 MLOps 과정에서 심화합니다.

---

### ✅ Day 5 최종 체크포인트

```
Q1. 전처리 파라미터(mean, std)를 모델과 함께 저장해야 하는 이유는?
모델 가중치가 "정규화된 입력"을 전제로 학습됐기 때문입니다.추론 시 입력을 학습 때와 똑같은 mean·std로 정규화해야 같은 기준으로 동작합니다. 이 값이 없으면 원본 입력이 그대로 들어가 학습 때와 완전히 다른 스케일이 되고, 예측이 무의미해집니다. 그래서 모델과 전처리 파라미터는 한 세트로 묶어 저장해야 합니다.
Q2. HousingRequest에서 Latitude에 ge=32, le=42를 넣은 이유는?
캘리포니아의 실제 위도 범위(약 32°~42°N)를 벗어난 입력을 검증 단계에서 막기 위해서입니다.모델은 캘리포니아 범위 데이터로만 학습했으므로, 범위 밖 값은 예측이 무의미합니다. 잘못된 입력은 추론까지 가기 전에 422 에러로 즉시 거부하는 게 안전합니다. 도메인 지식을 Pydantic 검증 규칙으로 옮겨 담은 것이죠.
Q3. Streamlit의 입력값 이름이 Pydantic 스키마의 필드 이름과 일치해야 하는 이유는?
JSON은 순서가 아니라 키(이름)로 값을 매칭하기 때문입니다.
FastAPI는 받은 JSON의 키를 HousingRequest 필드 이름과 하나씩 대조해 값을 채웁니다. 이름이 다르면(대소문자 포함) 필드를 못 찾아 **검증 실패(422)**가 납니다. 프론트(Streamlit)와 백엔드(Pydantic)가 "이름"이라는 약속으로 연결되어 있어서, 글자 하나까지 정확히 일치해야 합니다.
Q4. 이 프로젝트에서 run_in_executor를 제거하면 어떤 문제가 생길 수 있습니까?
추론이 이벤트 루프를 붙들어 동시 요청과 헬스체크가 막히는 문제가 생깁니다.
run_in_executor 없이 async def 안에서 추론을 직접 돌리면 루프(일꾼 한 명)를 점유해, ① 다른 요청이 순차 대기하고 ② /health까지 막혀 로드밸런서가 서버를 퇴출시킬 수 있습니다.

다만 주택 회귀 모델은 가벼워서 단일 요청에선 차이를 못 느낄 수 있습니다. 그래도 동시 요청이 몰리면 문제가 드러나므로, 부하 상황에서 무너지지 않는 토대로서 필요합니다.
Q5. MNIST 프로젝트(Day 1~4)와 오늘 프로젝트의 가장 큰 차이는 무엇입니까?
데이터 종류와 문제 유형이 다릅니다 — 이미지 분류 → 정형 데이터 회귀.
구분 | MNIST (Day 1~4) | 주택 가격 (Day 5)
데이터 | 이미지 (28×28 픽셀) | 정형 데이터 (숫자 특성들)
문제 유형 | 분류 (0~9) | 회귀 (연속된 가격값)
입력 방식 | 이미지 업로드 | 숫자 입력 폼
전처리 | 픽셀 정규화 | 특성별 mean/std 정규화
가장 본질적인 차이는 "이미지 분류"에서 "정형 데이터 회귀"로 바뀐 것이고, 이에 따라 입력 방식·전처리·검증 규칙이 모두 달라졌습니다. 핵심은 Day 1~4에서 익힌 배포 기술(FastAPI·비동기·검증·Streamlit)은 그대로 재사용하면서, 데이터·모델만 새 도메인으로 갈아끼웠다는 점입니다.
```

![image](model05_images/img06.png)

![image](model05_images/img07.png)

---

### 📌 Day 5 요약

```
오늘 한 일:
  ✅ 캘리포니아 주택 가격 데이터를 탐색하고 전처리 파이프라인을 구성했습니다.
  ✅ PyTorch 회귀 모델을 학습하고 전처리 파라미터와 함께 저장했습니다.
  ✅ FastAPI로 추론 API를 구축했습니다 (Pydantic 검증 + 비동기 처리).
  ✅ Streamlit으로 입력 폼 대시보드를 만들고 API와 연결했습니다.
  ✅ 통합 테스트로 정상/에러/동시 요청을 검증했습니다.
  ✅ Day 1~4의 모든 기술을 하나의 서비스에 통합하는 경험을 했습니다.
```

### 제출

다음 내역을 MD 파일 혹은 노트북 하단에 기록하여, 깃헙에 업로드하여 링크로 제출하시기 바랍니다  
> [제출 링크](https://forms.gle/vwiF5eX7PPB4snzr6)

1. 프로젝트 실행 내역 캡쳐와 설명
2. 각 섹션 체크포인트의 답변
3. 프로젝트 회고

수고하셨습니다!