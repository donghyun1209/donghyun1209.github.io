# 3주차 노트 (11/1~11/7) · 양자화 이론 + INT8 만들기

## 핵심 공식
```
양자화:   q     = clip(round(x / scale) + zero_point, qmin, qmax)
역양자화: x_hat = (q - zero_point) * scale
```
- **scale**: 정수 1칸이 실수로 얼마인지 = 실수 범위 폭 / 정수 범위 폭
- **zero_point**: 실수 0(정확히는 범위 시작값)이 어느 정수에 대응하는지
- 최대 오차 ≈ **scale / 2**: round는 가장 가까운 눈금으로 옮기므로 최악이 두 눈금의 딱 중간(반 칸). 평균은 그 절반(≈ scale / 4)
- → **범위를 좁게 잡을수록 scale이 작아지고 정밀해짐**. 아래 개념이 전부 이 한 줄로 설명됨

## 개념 4가지
| 개념 | 뜻 |
|---|---|
| PTQ vs QAT | PTQ: 학습 끝난 모델을 사후 변환. QAT: 학습 중 양자화 오차를 흉내 내며 학습 |
| 대칭 vs 비대칭 | 대칭: `[-max, +max]`, zero_point=0. 비대칭: `[min, max]`, zero_point≠0 |
| per-tensor vs per-channel | 텐서 전체에 scale 1개 vs 출력 채널마다 scale |
| calibration | activation 범위는 데이터를 흘려야 알 수 있음 → 대표 이미지로 범위 측정 |

## numpy로 직접 돌려 본 결과 (`quant_basics.py`)
x = -1~3 균등분포 1000개 (float32)

| 실험 | scale | zero_point | max error | 메모 |
|---|---|---|---|---|
| 비대칭 uint8 (`quant_asym(x)`) | 0.0157 | 64 | 0.0078 | = scale / 2 |
| 대칭 int8 (`quant_sym(x)`) | 0.0236 | 0 | 0.0118 | -3~-1 구간(약 1/3 칸)을 낭비해서 1.5배 |
| 비대칭 + 이상치 100 하나 (`quant_asym(x2)`) | 0.396 | 3 | 0.198 | 값 1개 때문에 **25배**. -1~3이 256칸 중 **10칸**에 몰림 |
| Percentile 99.9 (`quant_pct(x2)`) | 0.0156 | 64 | 전체 97.0 / 999개 **0.0078** | 이상치 1개는 희생(clip), 나머지는 이상치 없을 때 수준 |

### 대칭이 손해인데도 가중치에 쓰는 이유
- 실제 가중치는 0 중심으로 좌우가 비슷해서 낭비가 거의 없음
- zero_point=0이면 행렬곱에서 빼기 항이 사라짐 → NPU가 더 빠름
- YOLOv8 activation은 SiLU 뒤라 대부분 양수(최솟값 ≈ -0.28) → 비대칭이 유리

### calibration 방식
| 방식 | 범위 잡는 법 | 이상치 |
|---|---|---|
| MinMax | 진짜 min~max | 끌려감 → 위 25배 결과 |
| Percentile | 극단 0.1% 등은 무시 | 소수 희생, 다수 정밀 |
| Entropy | 양자화 전후 분포가 가장 비슷한 범위 탐색 | 자동으로 적당히 자름 |

## RasEyes 연결
- RasEyes INT8 변환에서 mAP가 떨어졌다면 activation 이상치 → MinMax calibration이 1순위 의심. rknn-toolkit2도 `quantized_algorithm`(normal / kl_divergence 등) 옵션이 있음
- calibration 이미지는 실제 카메라 환경과 비슷해야 함: 범위를 실제 입력 기준으로 재야 하니까

---

## INT8 모델 만들기 (`quantize.py`)
calibration 이미지: coco128 `train2017` 중 50장. 2주차 `preprocess()`를 그대로 import해서 사용 → 추론 때와 같은 전처리가 보장됨

### CalibrationDataReader
```python
class quantize(CalibrationDataReader):
    def __init__(self, image_paths):
        self.it = iter(image_paths)        # 리스트 → 하나씩 꺼내는 이터레이터
        self.input_name = "images"         # Netron에서 본 입력 이름

    def get_next(self):
        path = next(self.it, None)         # 다 꺼내면 None
        if path is None:
            return None
        return {self.input_name: preprocess(cv2.imread(path))[0]}
```
- `quantize_static`과의 약속: **`get_next()`를 계속 부르다 `None`이 나오면 멈춤**. 상속은 이 약속을 지킨다는 표시
- `__init__` = `get_next`가 쓸 재료(`self.it`, `self.input_name`)를 미리 만들어 두는 곳. `self.`를 붙여야 다른 메서드에서 보임
- `iter(리스트)` → `next(it, None)`: 하나씩 꺼내고, 바닥나면 기본값 `None`
- `preprocess()`는 `(텐서, scale, pad)` 튜플 반환 → `[0]`으로 텐서만

### quantize_static 옵션
| 옵션 | 값 | 이유 |
|---|---|---|
| `quant_format` | `QDQ` | Quantize/Dequantize 노드를 끼워 넣는 방식. 기본, 다른 백엔드로 옮기기 쉬움 |
| `per_channel` | `False` | per-tensor (→ 아래 남은 문제 후보) |
| `activation_type` | `QUInt8` | SiLU 뒤라 대부분 양수 → 비대칭 unsigned |
| `weight_type` | `QInt8` | 0 중심 분포 → 대칭 |
| `nodes_to_exclude` | `["/model.22/Concat_3"]` | 아래 함정 참고 |

- 크기: **13MB → 3.4MB** (≈ 1/4, float32 4바이트 → int8 1바이트)
- 실행 시 `Please consider to run pre-processing` 경고: 이미지가 아니라 **모델** 전처리(`quant_pre_process`: shape 추론·노드 합치기) 권고. 에러 아님

## FP32 vs INT8 mAP (`eval_map.py`, coco128 128장)
```python
YOLO("yolov8n-int8.onnx").val(data="coco128.yaml", imgsz=640, batch=1)
```
- mAP 계산은 직접 안 짜고 ultralytics `val()` 재사용 → `.onnx`도 그대로 받음
- 반복 단위는 **모델 개수(2)**. 이미지 128장은 `val()` 안에서 알아서 돎
- `batch=1`: export한 ONNX 입력이 `(1, 3, 640, 640)` 고정

| 모델 | mAP50-95 | mAP50 |
|---|---|---|
| FP32 | 0.452 | 0.606 |
| INT8 (Concat 포함 양자화) | **0.0** | **0.0** |
| INT8 (`Concat_3` 제외) | 0.367 | 0.557 |
| 차이 (FP32 → 최종 INT8) | −0.085 (약 19%↓) | −0.049 |

## 오늘 밟은 함정: 출력 Concat까지 양자화 → 점수가 전부 0
**증상**: INT8 mAP가 0.0

**원인 (자 비유)**
- INT8 = 눈금 256개짜리 자. per-tensor = 텐서 하나에 자 하나
- YOLO 출력 `(1, 84, 8400)` 한 텐서 안에 박스 좌표(0~640)와 점수(0~1)가 같이 들어 있음 (마지막 Concat이 합친 것)
- 640까지 재야 하니 눈금 한 칸 = 640 ÷ 256 ≈ **2.5**
- 점수 0.89는 첫 눈금(2.5)의 절반도 안 됨 → 가장 가까운 눈금 **0**으로 기록
- 점수가 전부 0 → 임계값 0.25를 넘는 박스 없음 → mAP 0

**확인 (bus.jpg 출력)**

| | 박스 최댓값 | 점수 최댓값 |
|---|---|---|
| FP32 | 637.3 | 0.89 |
| INT8 | 641.3 (멀쩡) | **0.0** (전멸) |

**해결**: 합치는 노드만 양자화에서 제외 → `nodes_to_exclude=["/model.22/Concat_3"]`. 합치기 전까지는 박스·점수가 각자 범위에 맞는 자를 씀
- 노드 이름 찾기: Netron에서 `output0` 바로 위 Concat 클릭

**교훈**: mAP가 이상하면 FP32와 INT8의 출력값 범위부터 나란히 찍어 보기

## 남은 문제
- −0.085는 정상 범위(0.01~0.03)보다 큼. mAP50보다 mAP50-95가 더 떨어짐 → **박스는 찾지만 위치가 덜 정확**
- 의심 1: head의 박스 디코딩(`Add`/`Sub`/`Div`/`Mul`, 0~640 범위)이 아직 양자화된 상태 → 제외해서 비교
- 의심 2: `per_channel=False` → `True`로 바꿔 비교
- 의심 3: `quant_pre_process` 미적용, MinMax calibration → `calibrate_method`(Percentile·Entropy) 비교

## RasEyes 연결 (INT8)
- `CalibrationDataReader` = RKNN `rknn.build(do_quantization=True, dataset='dataset.txt')`의 `dataset.txt` 역할
- RKNN 공식 YOLOv8 예제가 출력을 박스·점수로 **따로** 내보내는 것도 같은 이유: 범위가 크게 다른 값을 한 텐서로 양자화하지 않으려고 (RasEyes 출력 구조 다시 확인해 볼 것)
- RasEyes INT8에서 점수가 이상하게 낮았다면 1순위 확인 대상: 출력 텐서에 박스·점수가 함께 묶여 양자화됐는지

---

## 📖 논문: Designing Extremely Memory-Efficient CNNs for On-device Vision Tasks
Jaewook Lee, Yoel Park, Seulki Lee · ACCV 2024 (이슬기 교수님 UNIST 시절, 현 KAIST Embedded AI Lab. IJCV 2025 확장판 추정)
읽은 범위: 결론(p.14) + 3장 방법(p.4~9). 5장 한계(p.13~14)는 4주차에

### 한 줄 요약
MobileNetV2 bottleneck 기반 CNN을 **피크 메모리 63KB**(int8)로 돌림. ImageNet top-1 61.58%, PASCAL-VOC mAP 50.7%(65KB). STM32H7(MCU, RAM 256KB급)에서 실제 실행

- **피크 메모리** = 추론 중 한 순간에 들고 있는 양의 최댓값. 레이어마다 `입력 + 출력`이고, 그중 가장 큰 레이어가 피크
  - conv는 출력을 다 만들기 전엔 입력을 못 지움 (이웃 출력 픽셀들이 입력 3×3 영역을 겹쳐 읽음)
  - 레이어가 끝나면 입력은 버리고 출력이 다음 입력이 됨 → 레이어별 막대(Fig.1)
- MobileNetV2: 3번째 레이어에서 5.6MB → 256KB MCU엔 안 들어감

### 설계 원칙 3개 (전부 "한꺼번에 들고 있는 양 줄이기", 계산량 줄이기 아님)
| 원칙 | 하는 일 | 메모리 | 대가 |
|---|---|---|---|
| input segmentation (3.1) | 224×224를 k=4 조각 + 가운데 조각, 여백 m=18로 겹치게 | 조각 하나 ≈ 원본의 0.337 | 조각 경계의 맥락 손실 → 정확도↓ |
| patch tunneling (3.2) | 조각마다 독립 터널로 끝까지, 마지막에 **더하기**로 합침 | 전체 이미지 크기 feature map을 든 순간이 없음 → ≈ 1/k | 경계 걸친 물체를 중간 레이어에서 못 엮음 |
| bottleneck reordering (3.3) | expand→depthwise→reduce를 **채널 하나씩** 끝까지 흘리고 출력에 누적 | 중간값이 항상 1채널 분량 | 결과는 완전히 같음. 대신 병렬성↓, 반복 횟수↑ |

### 3.1 숫자로 확인
조각 비율 = `1/k + 2m/(h√k) + m²/h²` (h=224, k=4, m=18)
- = 0.25 + 0.087 = **0.337** (= 130² / 224²)
- 0.087은 performance loss가 아니라 **여백 때문에 늘어난 메모리 비용**. 정확도 손실을 줄이려고 일부러 냄
- 근거: 여백 0 → 61.31%, 여백 18 → 63.84% (Table 6) / 가운데 조각 빼면 63.84% → 62.32% (Table 4)

**피크 메모리 vs 계산량** (가운데 조각 추가 시)
| | 기준 | 가운데 조각 추가 |
|---|---|---|
| 피크 메모리 | 한 순간의 **최댓값** | 조각을 하나씩 처리, 크기도 같음 → 그대로 |
| 계산량 | 모든 순간의 **합** | 조각 4 → 5개. 0.337 × 5 ≈ 원본의 1.68배 |

비유: 책상(메모리)은 제일 큰 책 한 권만큼이면 되지만, 읽는 시간(계산량)은 책 수만큼 늘어남

### 3.2 patch tunneling 
```
조각1 (130×130) → [터널1: bottleneck 여러 개] → 작은 벡터 v1 ─┐
조각2           → [터널2]                    → v2 ─┤
...                                                ├→ v1+…+v5 → FC → 분류
가운데 조각      → [터널5]                    → v5 ─┘
      ↑ 터널 사이에 벽: 중간에 정보 안 주고받음
```
- 실행: 터널1을 **끝까지** 돌려 v1만 남기고 중간 feature map은 버림 → 터널2 … → 마지막에 벡터 5개를 더함
- 어느 순간에도 조각 크기 feature map만 메모리에 있음. 끝에 남기는 벡터는 작아서 무시 가능
- **합치기를 끝까지 미루는 게 핵심**: 레이어1 뒤에 바로 다시 이어 붙이면 큰 feature map이 부활해서 자른 의미가 없음
- 합칠 때 concat 대신 **더하기** (터널 끝 벡터 길이 3이라 치면)
  ```
  v1=[1,0,2]  v2=[0,3,1]  v3=[2,1,0]  v4=[0,0,1]  v5=[1,1,1]
  concat (이어 붙이기): [1,0,2, 0,3,1, 2,1,0, 0,0,1, 1,1,1]  → 길이 15 (5배)
  sum    (같은 칸끼리): [4,5,5]                              → 길이 3 (그대로)
  ```
  - 길이가 그대로면 메모리도, 다음 FC 레이어 가중치 수(입력 길이 × 클래스 수)도 그대로
  - 대신 concat은 "몇 번째 조각에서 온 값인지"가 자리로 남는데, sum은 섞여서 사라짐 → 아래 "위치 관계를 섞어 버림"과 같은 얘기
- 정확도 대가: 깊은 레이어일수록 한 픽셀이 보는 원본 범위(**receptive field**)가 넓어져야 "눈+눈+코 = 얼굴"을 만드는데, 터널은 아무리 깊어도 자기 조각 밖을 못 봄. 경계에 걸친 물체는 맨 끝 더하기에서 처음 만나고, 더하기는 위치 관계를 섞어 버림 → 여백·가운데 조각으로 보완
- RasEyes 비유: 640×640 대신 320×320 조각 4개를 작은 YOLO 4개로 따로 돌리고 끝에서 결과만 합치는 셈. 화면 가운데를 가로지르는 사람은 각 YOLO가 반쪽씩만 봄

**k를 늘리면?** (h=224, m=18)
| k | 조각 한 변 | 조각 비율 | 이상적 1/k 대비 | 전체 계산량 (k+1)×비율 |
|---|---|---|---|---|
| 4 | 112+18 = 130 | 0.337 | 1.35배 | ≈ 1.7배 |
| 16 | 56+18 = 74 | 0.109 | 1.75배 | ≈ 1.85배 |
- 메모리는 줄지만, **여백이 차지하는 비중이 커져서** 줄어드는 폭이 점점 작아짐
- 조각 하나가 보는 범위가 좁아져 정확도↓ (p.14 "Larger Images": 조각이 작을수록 한 조각의 정보가 줄고, 끝에서 합치기가 더 어려워짐)
- 정확도를 지키려고 여백을 키우면 그만큼 메모리 이득을 다시 깎아 먹음 → 공짜가 아님

### 3.3 reordering 숫자 예시 (픽셀 1개, 채널 3개)
입력 x=2, expand 가중치 a=(1,2,3), depthwise ×1, reduce 가중치 b=(1,2,1)
```
방법 A (원래): Y=(2,4,6) 전부 저장 → 1×2 + 2×4 + 1×6 = 16       중간값 3개 동시 보유
방법 B (reorder): acc=0
  채널1: Y1=2 → acc += 1×2 → 2   (Y1 버림)
  채널2: Y2=4 → acc += 2×4 → 10  (Y2 버림)
  채널3: Y3=6 → acc += 1×6 → 16  (Y3 버림)                       중간값 1개만 보유
```
- 값은 같고 메모리만 다름. 채널 200개면 A는 200개, B는 여전히 1개
- 가능한 이유: ① expand 채널 j는 입력 x만 필요 ② **depthwise는 정의상 채널별 독립** (일반 3×3 conv였으면 불가) ③ reduce는 결국 더하기라 나눠 더해도 같음
- Table 3: t(expansion ratio)를 2→8로 키우면 top-1 58.82% → 63.84%, 메모리는 254KB 그대로. 하나씩 계산하고 버리니까 채널이 늘어도 피크는 1채널 분량. 대신 반복 횟수(시간)와 가중치 수가 늘어남

### 3.3 reordering 자세히 (실제 크기)
MobileNetV2 중간 블록 하나, 입력 `56×56×24`, t=6
```
X  (입력)        56×56×24   =  75,264
 ↓ ① 1x1 expand (24 → 144)
Yp (expand 출력) 56×56×144  = 451,584   ← 큼
 ↓ ② 3x3 depthwise (채널별)
Yd (dw 출력)     56×56×144  = 451,584   ← 큼
 ↓ ③ 1x1 reduce (144 → 24)
Yo (출력)        56×56×24   =  75,264
```

**수식 (3)의 네 항**
| 항 | 원래 | reordering |
|---|---|---|
| X | 75,264 | 75,264 |
| Yp | 56×56×**144** = 451,584 | 56×56×**1** = 3,136 |
| Yd | 56×56×**144** = 451,584 | 56×56×**1** = 3,136 |
| Yo | 75,264 | 75,264 |
| 합 | 1,053,696 | 156,800 (≈ 1/6.7) |
- 수식 (3)의 회색 `1` 두 개 = 위 표의 144 → 1. t=8이면 원래는 192, reordering은 여전히 1 → Table 3

**Fig.5 한 사이클 (채널 j)**
```
① Yp_j = X * F_j^p        X 전체(24채널) 읽어서 56×56 한 장
② Yd_j = Yp_j * F_j^d     그 한 장만 3x3 depthwise
③ Yo  += Yd_j × (F^o의 j번째 줄)   → 출력 24채널 "각각에" 자기 몫을 더함
   Yp_j, Yd_j 버림 ("two intermediate outputs are freed")
j = 1 … 144 반복
```
- 성립 조건: ① X는 끝까지 들고 있음 ② depthwise·BN·ReLU6 모두 채널별 연산 ③ 1x1 conv 출력 = Σ(채널 j 값 × 가중치)라 나눠 더해도 같음

**대가**
- 큰 행렬곱 1번 → 작은 계산 144번: NPU·GPU처럼 한 번에 많이 하는 하드웨어에서 손해 큼
- Yo(75,264개)를 144번 읽고·더하고·다시 씀, X도 144번 다시 읽음 → 계산량은 같아도 메모리 접근↑
- 단일 코어 MCU(STM32)는 원래 하나씩 계산하는 구조라 손해가 상대적으로 작음 → 이 논문 타깃에선 거래가 맞음

### 세 원칙이 맞물리는 방식
| 원칙 | 줄이는 축 | 효과 범위 |
|---|---|---|
| segmentation + tunneling | **가로×세로** (224 → 130) | X·Yp·Yd·Yo 전부, 모든 레이어 |
| bottleneck reordering | **채널** (expand로 불어난 144 → 1) | Yp·Yd만 |
- reordering만: X·Yo가 전체 이미지 크기로 남음 / 자르기만: 가운데 항이 144채널로 남음 → **둘 다 써야 63KB**
- X·Yo의 채널 수(24)는 원칙이 줄이는 게 아니라 수식 (2)처럼 예산 안에 들어가게 설계할 때 고르는 값

## 내가 던진 질문: 그럼 추론 시간은?
- 논문엔 **latency 숫자가 없음** (표 1~9 전부 메모리·정확도만)
- reordering: 곱셈·덧셈 총량은 같지만 채널을 하나씩 → 병렬 하드웨어가 놂. 단일 코어 MCU는 손해 작음, NPU는 클 것
- segmentation + tunneling: 겹치는 여백·가운데 조각 때문에 계산량 자체가 ≈ 1.7배
- → 이 논문은 **시간을 내주고 메모리를 얻는** 거래. RAM 256KB면 성립, RAM 넉넉하고 FPS가 중요한 곳이면 방향이 반대

## RasEyes 연결 (재료, 한 줄은 4주차에 직접)
- Orange Pi 5는 RAM이 GB 단위라 메모리가 병목이 아님. RK3588 NPU는 채널 병렬 처리가 강점이라 reordering을 그대로 쓰면 오히려 느려질 것
- 가져올 수 있는 건 기법 자체보다 관점: **피크 메모리는 weight가 아니라 activation(중간 출력)에서 나온다**, 그리고 레이어별로 따져야 한다
- 양자화와의 관계: 254KB → 63KB = 정확히 1/4. 이번 주 INT8(13MB → 3.4MB)과 같은 이유. 논문 p.3도 "양자화로 줄일 수 있는 건 최대 4배"라고 함 → 그 이상은 구조를 바꿔야 함
