---
title: "4주차 노트 (11/18~11/19) · 지연·메모리 프로파일링"
date: 2026-11-19 09:00:00 +0900
categories: [Edge AI, YOLO 경량화]
tags: [onnxruntime, int8, profiling, latency, quantization, yolov8, sqnr, rknn, amdahl]
---

이번에는 "그래서 얼마나 빨라졌고, 정확도는 어디서 떨어졌나"를 직접 재 보기로 했다.\\
This time I set out to measure "so how much faster is it, and where did the accuracy drop?"

---

## 측정해 보니 2.3배 빨라졌다

측정 방법은 단순하게 잡았다. 처음 10번은 warm-up으로 하고, 그다음 100번을 재서 평균과 p90을 봤다. p90은 100번 중 90등의 값인데, 실제 상황에서는 통계의 함정이 있을 수 있기 때문에 확인했다.\\
I kept the measurement simple. The first 10 runs are warm-up, then measured 100 runs and looked at the mean and p90. P90 is the 90th value out of 100, I checked it because there can be statistical pitfalls in real situations.

결과는 FP32가 평균 52.4ms, INT8이 22.8ms로 약 2.3배 빨라졌다. 평균과 p90이 거의 같아서 튀는 값도 없었다. 다만 처음 돌렸을 때는 FP32 30.9ms, INT8 16.5ms가 나왔다. 숫자 자체는 CPU 클럭이나 절전 상태에 따라 실행마다 크게 바뀌니, 같은 실행 안에서의 **비율**만 믿기로 했다.\\
FP32 averaged 52.4ms and INT8 22.8ms — about 2.3× faster. The mean and p90 were almost identical, so there were no spikes. But on my first run, I got FP32 30.9ms and INT8 16.5ms. The absolute numbers change a lot between runs depending on CPU clock and power-saving state, so I decided to trust only the **ratio** within the same run.

QDQ 모델은 파일만 보면 `Quantize → Dequantize → Conv(FP32)` 순서라서 오히려 느려야 정상이라고 생각 할 수 있다. 그런데 ONNX Runtime이 세션을 만들 때 이 패턴을 `QLinearConv` 같은 정수 연산 노드 하나로 합쳐 주기 때문에 실제 계산은 int8로 돈다.\\
Looking at the file alone, a QDQ model runs `Quantize → Dequantize → Conv(FP32)`, which should actually be slower. But ONNX Runtime fuses this pattern into a single integer node like `QLinearConv` when it builds the session, so the real computation runs in int8.

---

## 4배가 아니라 2배인 이유는?

크기가 1/4이 됐으니 속도도 4배쯤 빨라질 줄 알았다. 처음엔 "일부 연산이 여전히 float로 돌고, int8과 float를 오가는 변환 비용 때문이겠지" 하고 넘기려 했는데, 실제로 재 보니 float 연산은 합쳐서 0.1ms, 변환은 0.4ms뿐이었다. 전체 18.5ms 중에서 말이다. 예상이 틀렸으니 연산별로 쪼개서 보기로 했다.\\
Since the size shrank to 1/4, I expected roughly 4× speed too. At first I was going to brush it off as "some ops still run in float, plus the cost of converting between int8 and float," but when I measured it, float ops totaled 0.1ms and conversions only 0.4ms — out of 18.5ms. My guess was wrong, so I decided to break it down per operation.

ONNX Runtime은 `SessionOptions`에서 `enable_profiling = True`만 켜면 노드별 시간을 JSON으로 남겨 준다. 세션 생성부터 전부 기록되기 때문에 warm-up 구간은 빼고, 추론마다 하나씩 찍히는 `model_run` 항목을 기준으로 측정 구간의 노드 시간만 연산 종류별로 더했다.\\
ONNX Runtime writes per-node timings to JSON once you set `enable_profiling = True` in `SessionOptions`. Since it records everything from session creation, I skipped the warm-up section and, using the `model_run` entry logged once per inference, summed only the node times in the measured section by op type.

| 연산 | FP32 (ms) | INT8 (ms) | 변화 |
|---|---|---|---|
| Conv | 37.0 | 8.2 | 4.5배↓ |
| Softmax | 7.0 | 0.8 | 9배↓ |
| SiLU | 3.2 | 4.6 | **느려짐** |
| Concat | 1.2 | 2.2 | 느려짐 |
| Resize | 0.5 | 1.3 | 느려짐 |
| 합계 | ≈ 52 | ≈ 18.5 | ≈ 2.8배↓ |

Conv는 곱셈과 덧셈이 잔뜩 있는 compute-bound 연산이라 int8로 한 번에 4배 많은 숫자를 처리하니 거의 4배 그대로 빨라졌다. 반면 SiLU, Concat, Resize는 값마다 계산은 한두 번뿐이고 시간 대부분을 메모리 읽고 쓰기에 쓰는 memory-bound 연산이라 int8 이득이 작았다. 오히려 느려지기까지 했다.\\
Conv is compute-bound, full of multiplies and adds, so with int8 processing 4× more numbers at once it got nearly 4× faster. SiLU, Concat, and Resize, on the other hand, do only one or two calculations per value and spend most of their time reading and writing memory — memory-bound — so int8 helped little. They even got slower.

SiLU가 느려진 이유가 재미있었다. FP32에서는 ORT가 `Sigmoid`와 `Mul`을 `QuickGelu` 하나로 합쳐서 한 번에 처리하는데, INT8에는 그런 합치기 규칙이 없어서 `QLinearSigmoid`와 `QLinearMul` 두 번으로 나뉘어 돈다. memory-bound 연산에서는 숫자 크기보다 몇 번에 나눠 읽고 쓰느냐가 더 중요하다는 걸 여기서 체감했다.\\
The reason SiLU got slower was interesting. In FP32, ORT fuses `Sigmoid` and `Mul` into a single `QuickGelu`, but INT8 has no such fusion rule, so it runs as two separate ops, `QLinearSigmoid` and `QLinearMul`. This is where I really felt that for memory-bound ops, how many passes you split reads and writes into matters more than the number size.

---

## 정확도는 한 곳이 아니라 조금씩 새고 있었다

정리하면 INT8은 크기 1/4, 속도 약 2.3배를 얻는 대신 mAP50-95가 0.452에서 0.367로 약 19% 떨어졌다. 전에 남긴 숙제가 "어느 레이어에서 떨어지나"였어서 찾아봤다.\\
To sum up, INT8 gains 1/4 the size and about 2.3× speed, but mAP50-95 dropped from 0.452 to 0.367, about 19%. Before work's homework was "which layer is it dropping at?", so I went looking.

방법은 같은 이미지 5장을 FP32와 INT8에 넣고 중간 출력을 하나씩 비교하는 것이다. 직접 짤 필요 없이 ORT에 이미 `qdq_loss_debug`가 있어서 그걸 썼다. 여기서 나오는 숫자는 두 가지다. `qdq_err`는 그 레이어 하나만 양자화했을 때 생긴 오차고, `xmodel_err`는 원래 FP32 모델과 비교한 오차, 즉 앞에서 쌓인 것까지 포함한 오차다.\\
The method was to feed the same 5 images into FP32 and INT8 and compare intermediate outputs one by one. No need to write it myself — ORT already has `qdq_loss_debug`. It gives two numbers. `qdq_err` is the error from quantizing just that one layer, and `xmodel_err` is the error compared to the original FP32 model, including everything accumulated before it. 

결과를 보니 가장 나쁜 곳은 head가 아니라 backbone 끝(`model.7~8`)과 neck(`model.18`, `model.21`) 같은 깊은 경로였다. `qdq_err`는 대부분 20~35dB로 괜찮은데 `xmodel_err`는 7~11dB였다. 어느 한 레이어가 망친 게 아니라, 레이어마다 작은 오차가 쌓인 것이다. 제일 확실한 예가 `Split`이었다. 텐서를 자르기만 하니 자체 오차는 거의 없는데(358dB), `xmodel_err`는 꼴찌였다. 이미 틀린 값을 그대로 넘겨받았을 뿐이다.\\
The worst spots weren't in the head but in deep paths like the end of the backbone (`model.7~8`) and the neck (`model.18`, `model.21`). `qdq_err` was mostly a fine 20–35dB, but `xmodel_err` was 7–11dB. No single layer broke it — small errors piled up layer by layer. The clearest example was `Split`. It only slices a tensor, so its own error is almost zero (358dB), yet its `xmodel_err` was the worst. It just passed along values that were already wrong.
