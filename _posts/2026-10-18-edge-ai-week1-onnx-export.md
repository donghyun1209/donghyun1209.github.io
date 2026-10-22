---
title: "1주차 노트 (10/18) · 몸풀기와 환경 세팅"
date: 2026-10-18 09:00:00 +0900
categories: [Edge AI, YOLO 경량화]
tags: [yolov8, onnx, onnxruntime, ultralytics, pytorch, nms, quantization, rknn]
---

## 환경 세팅과 export

`.venv` 가상환경을 만들고 torch(CPU), ultralytics, onnx, onnxruntime을 설치했다. GPU 없이 CPU 버전으로 충분한 단계라 설치는 금방 끝났다.\\
I created a `.venv` virtual environment and installed torch (CPU), ultralytics, onnx, and onnxruntime. The CPU version is enough at this stage, so installation finished quickly.

```python
from ultralytics import YOLO
yolo = YOLO("yolov8n.pt")               # 없으면 자동 다운로드
yolo.export(format="onnx", imgsz=640)   # 입력 크기 고정 → 양자화·RKNN 변환 때 덜 막힘
```

여기서 신경 쓴 건 `imgsz=640`으로 입력 크기를 고정한 것 하나다. 입력 크기가 동적이면 나중에 양자화나 RKNN 변환에서 막히기 쉬워서, 처음부터 고정해두는 쪽을 택했다.\\
The one thing I paid attention to was fixing the input size with `imgsz=640`. A dynamic input size tends to cause trouble later in quantization or RKNN conversion, so I chose to fix it from the start.

---

## 출력 `(1, 84, 8400)` 뜯어보기

출력 shape가 `(1, 84, 8400)`이고 하나하나 뜯어 다시 이해하기로 결정했다.\\
The output shape is `(1, 84, 8400)`, and I chose to break it down and understand it piece by piece.

84는 4 + 80이다. 앞의 4개는 박스의 `cx, cy, w, h`(640 입력 기준 픽셀)이고, 뒤의 80개는 COCO 80개 클래스 각각의 점수(0~1)다. YOLOv5는 여기에 objectness 값이 하나 더 있어서 85였는데, v8에서는 그게 빠진 것으로 알고 있다.\\
84 is 4 + 80. The first 4 are the box `cx, cy, w, h` (in pixels relative to the 640 input), and the remaining 80 are scores (0–1) for each of the 80 COCO classes. YOLOv5 had one extra objectness value, making it 85, but as far as I know v8 dropped it.

8400은 박스 후보의 개수다. 640을 stride 8, 16, 32로 나누면 80, 40, 20이 나오고, 격자 칸마다 후보를 하나씩 내니까 80² + 40² + 20² = 6400 + 1600 + 400 = 8400이 된다. 세 격자의 결과가 마지막에 Concat으로 이어 붙여진 것이다.\\
8400 is the number of box candidates. Dividing 640 by strides 8, 16, and 32 gives 80, 40, and 20, and each grid cell produces one candidate, so 80² + 40² + 20² = 6400 + 1600 + 400 = 8400. The results of the three grids are joined with a Concat at the end.

| 격자 (Grid) | stride | 잘 잡는 것 (Good at) |
|---|---|---|
| 80×80 | 8 | 작은 물체 (small objects) |
| 40×40 | 16 | 중간 물체 (medium objects) |
| 20×20 | 32 | 큰 물체 (large objects) |

후보 하나를 꺼내려면 `out[0, :, i]`로 84개 값을 가져오면 된다.\\
To pull out one candidate, take the 84 values with `out[0, :, i]`.

이걸 정리하고 나니 RasEyes 후처리 코드에서 출력이 왜 3개였는지가 더 잘 이해됐다.\\
After working this out, I understood better why the RasEyes post-processing code had three outputs.

---

## 다음 진행할 내용

다음에는 ultralytics에 기대지 않고 ONNX Runtime으로 추론을 직접 돌려볼 생각이다. letterbox, 정규화, NCHW 변환 같은 전처리를 손으로 짜고, 이번에 뜯어본 `(1, 84, 8400)` 출력을 박스 디코딩과 NMS로 직접 후처리해서 ultralytics 결과와 맞는지 비교해볼 것이다.\\
Next time I plan to run inference myself with ONNX Runtime without relying on ultralytics. I'll hand-write the preprocessing (letterbox, normalization, NCHW conversion), post-process the `(1, 84, 8400)` output I studied with box decoding and NMS, and compare the results against ultralytics.
