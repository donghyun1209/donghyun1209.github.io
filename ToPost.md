# 2주차 노트 (10/22~10/29) · ONNX Runtime 추론 직접 짜기

> 1주차가 일찍 끝나서 10/23부터 당겨서 시작

## 전처리 단계별 shape·dtype (bus.jpg 기준)
| 단계 | 코드 | shape | dtype / 값 |
|---|---|---|---|
| 읽기 | `cv2.imread` | `(1080, 810, 3)` HWC | uint8, **BGR** |
| letterbox | `letterbox(img)` | `(640, 640, 3)` | uint8, BGR |
| 색 순서 | `cv2.cvtColor(..., COLOR_BGR2RGB)` | `(640, 640, 3)` | uint8, RGB |
| 정규화 | `.astype(np.float32) / 255` | `(640, 640, 3)` | float32, 0~1 |
| 축 순서 | `np.transpose(..., (2, 0, 1))` | `(3, 640, 640)` CHW | float32 |
| 배치 축 | `np.expand_dims(..., axis=0)` | `(1, 3, 640, 640)` NCHW | float32 |

→ 최종 출력 `(1, 3, 640, 640) float32 0.0 1.0` = 1주차 Netron에서 본 입력 shape와 일치

## letterbox
- 비율 유지하며 긴 변을 640에 맞추고, 남는 칸은 회색(114)으로 채움 → 물체가 찌그러지지 않음
- `scale = min(size/h, size/w)`: 더 많이 줄여야 하는 쪽 기준 (max면 640을 넘어서 잘림)
- 패딩: `top = (size - new_h)//2`, `bottom = size - new_h - top` (좌우도 동일)

| 이미지 | 원본 (H, W) | scale | pad (x, y) |
|---|---|---|---|
| bus.jpg (세로) | 1080 × 810 | 0.593 | (80, 0) → 좌우 패딩 |
| zidane.jpg (가로) | 720 × 1280 | 0.5 | (0, 140) → 위아래 패딩 |

- **scale, pad를 같이 반환하는 이유**: 모델이 내놓는 박스는 640 letterbox 좌표 → 후처리에서 `(x - pad_x) / scale`로 원본 좌표로 되돌릴 때 필요

## 오늘 밟은 함정
- `cv2.resize(img, (너비, 높이))`: shape는 `(H, W)`인데 resize 인자는 `(W, H)` 순서
- 크기는 정수여야 함 → `round(w * scale)`
- cv2 함수(`resize`, `copyMakeBorder`, `cvtColor`)는 원본을 안 바꾸고 **새 결과를 반환** → 변수에 받아야 함
- `bottom = size - top` ✗ → 전체(640)가 아니라 **남는 칸**에서 top을 빼야 함. 남는 칸이 0이면 우연히 맞아 보여서 가로·세로 사진 둘 다로 확인해야 잡힘
- 함수가 값 3개를 반환하면 받을 때도 `a, b, c = f()`로 풀어서

## RasEyes 연결
- RasEyes에서는 `rknn.config()`에 mean/std를 넣어서 NPU가 정규화까지 처리 → ONNX Runtime엔 그런 기능이 없어서 직접 함
- 카메라 프레임 → letterbox → NPU 입력: RasEyes에서 하던 전처리를 이번에 손으로 다시 짠 것

---

## onnxruntime 추론
```python
sess = ort.InferenceSession("yolov8n.onnx", providers=["CPUExecutionProvider"])  # 한 번만
outputs = sess.run(None, {sess.get_inputs()[0].name: out})                      # 매 프레임
```
| 항목 | 값 |
|---|---|
| 입력 | `images`, float, `[1, 3, 640, 640]` |
| 출력 | `output0`, float, `[1, 84, 8400]` → 1주차 export 로그와 일치 |

- `sess.run(None, ...)`: `None` = 출력 전부. 반환은 **리스트** → 실제 배열은 `outputs[0]`
- RasEyes 대응: `load_rknn` + `init_runtime` = `InferenceSession`, `rknn.inference` = `sess.run`

### 1등 후보 (bus.jpg)
- 점수 0.89, 클래스 0(person), 640 좌표 `cx,cy,w,h = 518.6, 373.4, 82.6, 295.8`
- 오른쪽 끝 `cx + w/2 ≈ 560 = 640 - pad 80` → 원본 오른쪽 끝에 잘린 사람

### 추론 시간 (CPU, FP32)
| 모델 | 실행 환경 | 전처리 | 추론 평균 | 추론 최소 |
|---|---|---|---|---|
| yolov8n FP32 ONNX | CPU 8코어 (onnxruntime) | ~2 ms | **약 30~50 ms** (실행마다 다름) | ~29 ms |

- 워밍업 3회 → 20회 측정. 한 실행 안에서는 평균≈최소로 안정, **실행끼리** 30 ↔ 50ms로 흔들림
- 원인 후보: CPU 거버너 `powersave`(클럭 변동), 스레드 배치 → 벤치마크는 여러 번 돌려 **범위**로 기록

### ① 임계값 거르기
- `maxPoint = check2.max(axis=0)` 후보별 최고 점수, `check2.argmax(axis=0)` 후보별 **클래스 번호**
- `mask = maxPoint > 0.25` → True/False `(8400,)`, `mask.sum()` = 46
- 같은 mask를 점수·클래스·박스에 전부 적용 → 칸 번호(=후보)가 맞춰진 채로 걸러짐. 박스는 2차원이라 `pred[:4, mask]`
- 물체 5개인데 46개 → 이웃 격자 칸 + 다른 격자(80/40/20)가 같은 물체에 동시 반응 → **NMS가 필요한 이유**

### ② cx,cy,w,h → x1,y1,x2,y2
- `x1 = cx - w/2`, `x2 = cx + w/2`, y도 동일
- **이미지 y축은 아래로 갈수록 커짐** → 위쪽 = `cy - h/2`
- `np.stack([x1, y1, x2, y2], axis=1)` → `(46, 4)` 한 줄에 박스 하나
- 1등 박스 (640 좌표): `[477.3, 225.5, 560.0, 521.3]`, 0.89, person

### ③ letterbox 되돌리기
- 전처리가 `× scale → + pad`였으니 되돌리기는 **역순**: `- pad → ÷ scale`
```python
pad_x, pad_y = pad
xyxy[:, [0, 2]] -= pad_x      # x열(0, 2)
xyxy[:, [1, 3]] -= pad_y      # y열(1, 3)
xyxy /= scale
H, W = img.shape[:2]
xyxy[:, [0, 2]] = np.clip(xyxy[:, [0, 2]], 0, W)
xyxy[:, [1, 3]] = np.clip(xyxy[:, [1, 3]], 0, H)
```
- `xyxy[:, [0, 2]]`: 모든 박스의 0번·2번 열. `-=`를 붙여 원래 배열을 바꿈
- clip: 패딩 영역에 걸친 박스가 원본 밖(음수, 810 초과)으로 나감 → 잘라냄. 46개 기준 x 최소 0.0, 최대 810.0으로 실제로 잘림
- 1등 박스 (원본 좌표): `[670.4, 380.6, 809.9, 879.7]` → x2 ≈ 810 = 원본 너비 → 오른쪽 끝에 잘린 사람 확인

### ④ IoU
- **IoU = 겹친 넓이 / (넓이1 + 넓이2 − 겹친 넓이)**. 겹친 부분이 두 번 더해지니 한 번 뺌
- 겹친 영역: 왼쪽·위 = 두 박스 x1·y1 중 **max**, 오른쪽·아래 = x2·y2 중 **min**
- 박스 넓이 = `(x2 - x1) * (y2 - y1)` (가로 × 세로)
- 안 겹침 판정은 가로·세로를 **각각** 확인. 곱한 다음에 `max(0, 곱)`을 하면 음수 × 음수 = 양수라서 안 겹친 박스가 겹친 걸로 나옴

| 테스트 | 기대값 |
|---|---|
| 똑같은 박스 | 1.0 |
| 반만 겹침 `[0,0,10,10]` vs `[5,0,15,10]` | 50 / (100 + 100 − 50) = 0.333 |
| 대각선으로 떨어짐 | 0 |
| x로만 떨어짐 / y로만 떨어짐 | 0 (음수 IoU가 나오면 버그) |
| 안에 쏙 들어감 `[2,2,8,8]` | 36 / 100 = 0.36 |

### ⑤ NMS
```python
def nms(boxes, scores, iou_thres=0.45):
    h2l = list(np.argsort(scores)[::-1])     # 점수 높은 순 칸 번호
    keep = []
    while len(h2l) > 0:
        best = h2l[0]
        keep.append(best)
        h2l = [j for j in h2l[1:] if iou(boxes[best], boxes[j]) < iou_thres]
    return keep
```
- `np.argsort`는 **오름차순 칸 번호**를 돌려줌 → `[::-1]`로 뒤집기
- 리스트 컴프리헨션 `[j for j in 리스트 if 조건]`: 1등과 덜 겹친(다른 물체일 수 있는) 박스만 남김
- 결과 46 → **5개** (사람 4 + 버스 1)

### ⑥ ultralytics와 비교
| 내 코드 | ultralytics | 점수 | 클래스 |
|---|---|---|---|
| 670.4, 380.6, 809.9, 879.7 | 670.45, 380.56, 809.92, 879.65 | 0.890 | 0 person |
| 221.7, 407.4, 343.8, 856.2 | 221.68, 407.42, 343.78, 856.19 | 0.883 | 0 person |
| 50.7, 397.3, 244.4, 905.4 | 50.68, 397.30, 244.40, 905.38 | 0.878 | 0 person |
| 31.5, 230.7, 801.4, 775.8 | 31.49, 230.74, 801.41, 775.78 | 0.843 | 5 bus |
| 0.4, 549.8, 57.8, 868.3 | 0.45, 549.84, 57.77, 868.30 | 0.436 | 0 person |

- 같은 ONNX로 비교 (`YOLO("yolov8n.onnx", task="detect")`) → 전처리·후처리만의 차이를 보기 위해
- `6.7045e+02` = 6.7045 × 10² = 670.45 (지수 표기)
- `compare.py`는 남겨 둠 → INT8 모델 만들면 FP32와 비교할 때 재사용

### 클래스 무시 NMS vs 클래스별 NMS
- 내 NMS는 **클래스 무시**(agnostic): 사람과 버스도 IoU만 높으면 지움
- ultralytics 기본값은 `agnostic_nms=False` → **클래스별로 따로** NMS
- 이번엔 결과가 같았던 이유: 버스와 사람들의 IoU = 0.014 ~ 0.165로 0.45보다 훨씬 작음
  - 사람이 버스 앞에 있어도 버스가 커서 **합친 넓이(분모)**가 커지니 IoU가 작음
- 큰 물체 앞에 꽉 차게 서 있는 사람처럼 IoU가 높은 다른 클래스가 있으면 내 코드는 지워 버릴 수 있음

## 오늘 밟은 함정 (추론·후처리)
- 세션을 for 안에서 매번 생성 → 모델 로드 반복. 세션은 시작할 때 한 번만 (RasEyes도 `load_rknn`은 한 번)
- `np.clip`, `np.argsort`도 **새 결과를 반환** → `=`로 받아야 함 (cv2 함수와 같음)

## RasEyes 연결 (추론·후처리)
- RasEyes RKNN 출력은 격자 3개가 따로 나옴 → 이번 ONNX는 이미 Concat된 `(84, 8400)`. 디코딩 위치만 다르고 거르기·NMS 흐름은 같음
- Orange Pi에서 측정값이 흔들렸다면 클럭·발열 쓰로틀링 의심
