---
title: "RasEyes 개발일지: RKNN INT8 양자화 모델 빌드 및 NPU 벤치마크"
date: 2026-06-26 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [rknn, int8, npu, yolov8, github-actions, onnx, quantization, pytest, gpiod, python]
---

드디어 YOLOv8n을 INT8로 양자화해서 Orange Pi 5의 NPU에 올려봤습니다.\\
I finally quantized YOLOv8n to INT8 and got it running on the Orange Pi 5's NPU.

---

## 1. GitHub Actions가 세 번 연속으로 말썽을 부렸다 (GitHub Actions broke three times in a row)

로컬에서 RKNN 모델 변환하는 게 귀찮아서 `build_rknn.yml` 워크플로우를 만들어뒀는데, 돌릴 때마다 새로운 에러가 튀어나왔습니다.\\
I'd set up a `build_rknn.yml` workflow so I wouldn't have to convert the RKNN model locally every time — but every run threw a brand-new error.

**1차: 파라미터명이 달랐다.** rknn-toolkit2가 v2.x로 올라오면서 `quantization_algorithm`이 `quantized_algorithm`으로 이름이 바뀌어 있었습니다. 문서랑 실제 API가 따로 놀고 있었던 것입니다.\\
**Round 1: the parameter name had changed.** rknn-toolkit2's v2.x update had silently renamed `quantization_algorithm` to `quantized_algorithm`. The docs and the actual API had drifted apart.

**2차: onnx 버전이 문제였다.** `onnx>=1.16.1`을 그대로 뒀더니 빌드가 깨졌습니다. `onnx.mapping` 모듈이 1.17부터 아예 없어졌기 때문이었습니다. `onnx>=1.14.0,<=1.16.0`으로 못 박아서 해결했습니다.\\
**Round 2: it was the onnx version.** Leaving `onnx>=1.16.1` as-is broke the build, because the `onnx.mapping` module was removed entirely starting in 1.17. I pinned it to `onnx>=1.14.0,<=1.16.0` and moved on.

**3차: urllib이 리다이렉트를 못 따라갔다.** INT8 캘리브레이션용 COCO128 데이터셋을 받는 스텝에서 urllib이 308 리다이렉트를 처리하지 못해 다운로드가 실패했습니다. `wget`으로 바꾸는 걸로 끝냈습니다.\\
**Round 3: urllib couldn't follow the redirect.** The step downloading the COCO128 dataset for INT8 calibration failed because urllib doesn't handle 308 redirects. Switching to `wget` fixed it.

세 번의 사고를 다 넘기고 나서야 `yolov8n.rknn`(4.8MB)이 빌드됐고, `scp`로 보드에 옮겼습니다.\\
Only after clearing all three did `yolov8n.rknn` (4.8MB) finally build, and I scp'd it over to the board.

---

## 2. 보드에서 돌리니 또 두 개가 터졌다 (Running it on the board turned up two more)

모델은 빌드됐는데, `vision/rknn_detector.py`에서 돌려보니 이번엔 다른 버그 두 개가 기다리고 있었습니다.\\
The model was built, but running it through `vision/rknn_detector.py` turned up two different bugs.

하나는 입력 텐서 차원이었습니다. `(H,W,C)` 형태로 그냥 넘기고 있었는데, `np.expand_dims`로 배치 차원을 명시해서 `(1,H,W,C)`로 만들어줘야 했습니다.\\
One was the input tensor dimension. I was passing it straight in as `(H,W,C)`, but it needed the batch dimension made explicit via `np.expand_dims`, turning it into `(1,H,W,C)`.

다른 하나는 NPU 코어 설정 누락이었습니다. `init_runtime(core_mask=RKNNLite.NPU_CORE_0_1)`을 넣어주지 않으면 듀얼 코어 중 하나만 쓰고 있었습니다.\\
The other was a missing NPU core setting — without `init_runtime(core_mask=RKNNLite.NPU_CORE_0_1)`, it was only using one of the two NPU cores.
