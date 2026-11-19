# 4주차 노트 (11/18~11/19) · 지연·메모리 프로파일링

## 레이턴시 측정 (`bench.py`)
```python
def bench(model_path, x, warmup=10, runs=100):
    sess = ort.InferenceSession(model_path, providers=["CPUExecutionProvider"])
    name = sess.get_inputs()[0].name
    for _ in range(warmup):
        sess.run(None, {name: x})
    times = []
    for _ in range(runs):
        t0 = time.perf_counter()
        sess.run(None, {name: x})
        t1 = time.perf_counter()
        times.append((t1 - t0) * 1000)
    return times
```
- warm-up: 처음 몇 번은 메모리 할당·캐시 적재 때문에 느림 → 버리고 측정
- p90 = `np.percentile(times, 90)`: 100번 중 느린 쪽 10%가 시작되는 값. 실시간에선 평균보다 꼬리 지연이 끊김을 결정

| 모델 | mean | p90 |
|---|---|---|
| FP32 | 52.4 ms | 53.5 ms |
| INT8 | 22.8 ms | 23.0 ms |
| 배율 | **≈ 2.3배** 빠름 | |

- mean ≈ p90 → 튀는 값 없이 안정적
- 첫 실행에선 FP32 30.9 / INT8 16.5ms였음. **절대값은 실행마다 크게 바뀜**(CPU 클럭·절전·백그라운드) → 같은 실행 안의 **비율**만 비교할 것

### 왜 INT8이 빨라졌나
- QDQ 포맷은 파일상 `Quantize → Dequantize → Conv(FP32)` 순서. 그대로 돌면 오히려 느림
- ONNX Runtime이 세션 만들 때 그래프 최적화로 이 패턴을 `QLinearConv` 같은 **정수 연산 노드로 합침**(fusion) → 실제 계산은 int8

### 왜 4배가 아니라 2배대인가
- 4배는 **크기**(4바이트 → 1바이트) 얘기. 속도는 다름
- 처음 생각: "일부 연산은 여전히 float로 돌고, int8 ↔ float로 바꾸는 데도 시간이 들어서 그럴 것"
  - 실제로 재 보니 **아니었음**. float로 도는 연산은 합쳐서 0.1ms, 바꾸는 시간은 0.4ms뿐 (전체 18.5ms 중)

## ORT 프로파일링 (`profile_ort.py`)
```python
opts = ort.SessionOptions()
opts.enable_profiling = True                  # 노드별 시간 기록
sess = ort.InferenceSession(path, sess_options=opts, providers=[...])
# warm-up 3 + 측정 10
json_path = sess.end_profiling()              # JSON 파일 이름 반환
```
- 세션 만들 때부터 끝까지 전부 기록됨 (warm up은 제외하고 읽기)
  - 추론 1번마다 `"name": "model_run"` 항목 1개 → 4번째 `model_run`의 `ts` 이후 Node만 합침
- JSON에서 쓰는 키: `cat`("Node"만), `dur`(µs), `ts`(시작 시각), `args.op_name`(연산 종류)
- 집계: `total[op] = total.get(op, 0) + dur` → `sorted(total.items(), key=lambda kv: kv[1], reverse=True)`
  - `lambda kv: kv[1]` = (이름, 시간) 쌍에서 시간을 기준으로
- 메모리 정보도 있음: `activation_size`(그 노드 출력 크기). 예: Concat_3 = 1×84×8400×4바이트 = 2,822,400

### 연산별 FP32 vs INT8 (ms, 추론 1번)
| 연산 | FP32 | INT8 | 배율 |
|---|---|---|---|
| Conv | 37.0 | 8.2 | **4.5배↓** |
| Softmax | 7.0 | 0.8 | 9배↓ |
| SiLU | 3.2 (`QuickGelu` 1개) | 4.6 (`QLinearSigmoid` + `QLinearMul`) | **느려짐** |
| Concat | 1.2 | 2.2 | 느려짐 |
| Resize | 0.5 (`Upsample`) | 1.3 | 느려짐 |
| 레이아웃 변환 | 2.3 (`ReorderInput/Output`) | 0.5 (`Transpose`) | 줄어듦 |
| 합계 | ≈ 52 | ≈ 18.5 | ≈ 2.8배 |

- **Conv = compute-bound**: 곱셈·덧셈이 많음 → int8로 한 번에 4배 많은 숫자 처리 → 거의 4배 그대로
- **SiLU·Concat·Resize = memory-bound**: 값마다 계산 한두 번, 시간은 feature map 읽고 쓰기 → int8 이득 작음. **몇 번에 나눠 읽고 쓰느냐(fusion)**가 더 중요
  - FP32는 ORT가 `Sigmoid` + `Mul`을 `QuickGelu` 하나로 합침. INT8은 합치는 규칙이 없어 두 번 → 오히려 느림
- **암달의 법칙**: Conv 비중 71%(FP32) → 44%(INT8). 빨라진 부분이 줄수록 안 빨라진 부분이 전체를 결정
- SiLU(Sigmoid + Mul)가 INT8 시간의 **25%**. 계산량(FLOPs)은 Conv에 몰려 있어도 시간은 아님

## FP32 vs INT8 한눈에
| | FP32 | INT8 | 변화 |
|---|---|---|---|
| 지연 (mean, CPU) | 52.4 ms | 22.8 ms | ≈ 2.3배 빠름 |
| 크기 | 13 MB | 3.4 MB | ≈ 1/4 |
| mAP50-95 (coco128) | 0.452 | 0.367 | −0.085 (약 19%↓) |
| mAP50 | 0.606 | 0.557 | −0.049 |

- 지연은 이번 주 `bench.py`, mAP는 3주차 `eval_map.py` 결과 (INT8 = `Concat_3` 제외 버전)

## 레이어별 오차: 어디서 정확도가 새나 (`layer_error.py`)
3주차 숙제: mAP50-95가 19% 떨어졌고 박스 위치가 덜 정확함 → 어느 레이어에서?

**방법**: 같은 이미지(coco128 5장)를 FP32·INT8에 넣고 중간 출력을 하나씩 비교. ORT에 이미 있는 `qdq_loss_debug` 사용
```python
modify_model_output_intermediate_tensors("yolov8n.onnx", "yolov8n_modified.onnx")  # 중간값도 출력으로
fp32_acts = collect_activations("yolov8n_modified.onnx", ImageReader(paths))       # {텐서이름: [배열...]}
int8_acts = collect_activations("yolov8n-int8_modified.onnx", ImageReader(paths))
match = create_activation_matching(int8_acts, fp32_acts)   # QDQ 모델이 먼저
err = compute_activation_error(match)                      # {이름: {"qdq_err", "xmodel_err"}}
sorted(err.items(), key=lambda kv: kv[1]["xmodel_err"])[:10]
```
- 텐서 수: FP32 242 / INT8 612 (Q/DQ 노드 출력까지 포함)
- reader는 `iter()`라 한 번 읽으면 빔 → 모델마다 새로 만들기

**두 숫자 (SQNR, dB, 낮을수록 나쁨)**
- `qdq_err`: 그 레이어 **하나만** 양자화할 때 생긴 오차
- `xmodel_err`: FP32 모델과 비교한 오차 = **앞에서 쌓인 것까지** 포함
- 비유 (전화 게임): `qdq_err` = 들은 걸 얼마나 정확히 옮겼나 / `xmodel_err` = 처음 문장과 얼마나 다른가

**결과 (`xmodel_err` 나쁜 순 상위)**

| 텐서 | xmodel_err | qdq_err |
|---|---|---|
| `/model.21/Split` | 6.5 | 358.3 |
| `/model.18/Split` | 6.8 | 361.3 |
| `/model.8/cv2/act/Mul` | 7.1 | 31.6 |
| `/model.7/act/Mul` | 7.3 | 32.9 |
| `/model.8/Split` | 7.6 | 364.3 |
| `/model.21/cv2/act/Mul` | 7.7 | 30.3 |
| `/model.22/cv2.2/cv2.2.0/act/Mul` | 7.8 | 35.8 |
| `/model.22/cv2.1/cv2.1.1/conv/Conv` | 8.0 | 28.9 |

**읽은 것**
- 가장 나쁜 곳은 head가 아니라 **깊은 경로**(backbone 끝 `model.7~8`, neck `model.18`, `model.21`)
- `qdq_err`는 대부분 20~35 dB로 괜찮은데 `xmodel_err`는 7~11 dB → **한 레이어가 망친 게 아니라 레이어마다 작은 오차가 쌓인 것**
  - `Split`이 가장 깔끔한 예: 자르기만 해서 자체 오차 ≈ 0(358 dB)인데 `xmodel_err`는 꼴찌. 이미 틀린 값을 그대로 넘겨받음
- head(`model.22`) 안에선 박스 갈래 `cv2`(7.8~8.0)가 클래스 갈래 `cv3`(8.8~11)보다 조금 나쁨 → "mAP50-95가 더 떨어짐 = 위치가 덜 정확"과 일치

## RasEyes 연결
- RKNN 변환 로그의 "NPU 미지원 → CPU fallback" = 위 "정수 버전 없는 연산"과 같은 현상. NPU↔CPU 왕복 비용 때문에 기대만큼 안 빨라짐
- `rknn.build()`가 정수 연산으로 바꿔 주는 단계 ≈ ORT의 QDQ fusion
- rknn-toolkit2의 `accuracy_analysis()`가 위 `layer_error.py`와 같은 일(레이어별 FP32 vs 양자화 출력 비교)을 함. RasEyes INT8 정확도가 떨어지면 이걸로 쌓이는 지점부터 찾기
- NPU도 Conv는 압도적으로 빠르지만 SiLU·Concat 같은 memory-bound 연산은 비중이 커짐. NPU용 모델에서 SiLU → ReLU로 바꾸는 이유 중 하나: ReLU는 Conv에 합쳐서(fusion) 거의 공짜로 처리되기 쉬움
