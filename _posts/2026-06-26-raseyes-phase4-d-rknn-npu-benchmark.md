---
title: "RasEyes Phase 4-D: RKNN INT8 양자화 모델 빌드 및 NPU 벤치마크"
date: 2026-06-26 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [rknn, int8, npu, yolov8, github-actions, onnx, quantization, pytest, gpiod, python]
---

Orange Pi 5의 NPU에서 YOLOv8n INT8 양자화 모델을 빌드·배포하고, 실측 벤치마크로 추론 성능을 검증했습니다.\\
I built and deployed a YOLOv8n INT8 quantized model on the Orange Pi 5 NPU and validated inference performance through real-world benchmarks.

---

### RKNN 모델 빌드 자동화 (Phase 4-D-5)\\RKNN Model Build Automation (Phase 4-D-5)

`tests/benchmark_rknn.py`를 새롭게 작성하여 `RknnDetector` 전용 NPU 벤치마크를 구현했습니다.\\
I created a new `tests/benchmark_rknn.py` to implement an NPU benchmark dedicated to `RknnDetector`.

50회 연속 측정 후 평균/최소/최대/P95 지연과 FPS를 집계하고, KPI 기준치(지연 < 60ms, FPS ≥ 15)를 자동으로 판정하도록 했습니다.\\
It measures latency and FPS over 50 consecutive runs, then auto-evaluates against KPI thresholds (latency < 60ms, FPS ≥ 15).

GitHub Actions 워크플로우 `build_rknn.yml`에서는 세 가지 문제를 연속으로 디버깅했습니다.\\
Three issues were fixed in a row in the `build_rknn.yml` GitHub Actions workflow.

첫째, rknn-toolkit2 v2.x API 변경으로 파라미터명이 `quantization_algorithm`에서 `quantized_algorithm`으로 바뀐 것을 수정했습니다.\\
First, the parameter name change from `quantization_algorithm` to `quantized_algorithm` in the rknn-toolkit2 v2.x API was corrected.

둘째, `onnx>=1.16.1` 요구사항을 `onnx>=1.14.0,<=1.16.0`으로 제한했는데, `onnx.mapping` 모듈이 1.17 버전부터 제거되기 때문입니다.\\
Second, the `onnx` requirement was pinned to `>=1.14.0,<=1.16.0` because `onnx.mapping` was removed in version 1.17.

셋째, urllib의 308 리다이렉트 처리 누락 문제를 `wget` 교체로 해결하고, INT8 캘리브레이션용 COCO128 데이터셋을 자동 다운로드하는 스텝을 추가했습니다.\\
Third, urllib's failure to follow 308 redirects was resolved by switching to `wget`, and a step to automatically download the COCO128 dataset for INT8 calibration was added.

이 과정을 통해 INT8 양자화 모델 `yolov8n.rknn`(4.8MB)이 빌드 완료되었고, `scp`로 Orange Pi 5에 전송했습니다.\\
Through this process, the INT8 quantized model `yolov8n.rknn` (4.8MB) was successfully built and transferred to the Orange Pi 5 via `scp`.

---

### NPU 추론 속도 실측 (Phase 4-D-6)\\Real-World NPU Inference Speed (Phase 4-D-6)

`vision/rknn_detector.py`에서 두 가지 버그를 수정했습니다.\\
Two bugs were fixed in `vision/rknn_detector.py`.

첫 번째는 입력 텐서 차원 오류였습니다. `(H,W,C)` 형태로 넘기던 배열을 `np.expand_dims`로 `(1,H,W,C)`로 확장하여 배치 차원을 명시했습니다.\\
The first was an input tensor dimension error: the array passed as `(H,W,C)` was expanded to `(1,H,W,C)` via `np.expand_dims` to make the batch dimension explicit.

두 번째는 NPU 코어 설정 누락이었습니다. `init_runtime(core_mask=RKNNLite.NPU_CORE_0_1)`을 적용하여 듀얼 NPU 코어를 모두 활용하도록 수정했습니다.\\
The second was a missing NPU core configuration: `init_runtime(core_mask=RKNNLite.NPU_CORE_0_1)` was applied to fully utilize both NPU cores.

수정 후 서비스를 중단한 상태에서 50회 측정한 벤치마크 결과는 다음과 같습니다.\\
After the fix, benchmark results measured 50 times with the service stopped were as follows.

| 지표 | 수치 | KPI |
|------|------|-----|
| 평균 지연 | **27.5 ms** | < 60ms ✅ |
| 최소 지연 | 25.7 ms | - |
| 최대 지연 | 30.0 ms | - |
| P95 지연 | 29.9 ms | - |
| FPS | **36.3** | ≥ 15 ✅ |

FP16 모델이 56.7ms / 17.6 FPS를 기록했던 것과 비교하면 INT8 모델이 **2.1배** 빠른 추론 속도를 보입니다.\\
Compared to the FP16 model at 56.7ms / 17.6 FPS, the INT8 model is **2.1× faster**.

한 가지 주의할 점은, `raseyes.service` systemd 서비스가 실행 중인 상태에서는 NPU 경합이 발생하여 지연이 77ms까지 상승했습니다. 그러나 실제 운영 환경에서는 단일 프로세스만 NPU를 점유하므로 문제가 없습니다.\\
One caveat: when the `raseyes.service` systemd service was running, NPU contention pushed latency up to 77ms. In real operations, however, only a single process occupies the NPU, so this is not an issue.

---

### 코드 리뷰 및 리소스 안전성 강화\\Code Review and Resource Safety Improvements

벤치마크 검증 이후 코드 리뷰를 통해 리소스 누수와 예외 처리 취약점을 다수 발견하고 수정했습니다.\\
Following benchmark validation, a code review uncovered and resolved multiple resource leaks and exception handling weaknesses.

`vision/rknn_detector.py`에서는 `start()` 도중 예외가 발생할 때 NPU 핸들(`self._rknn`)이 해제되지 않는 누수를 수정했습니다. `stop()` 메서드에도 개별 `try-except`를 추가하여 NPU와 카메라 자원이 항상 강제 해제되도록 보장했습니다.\\
In `vision/rknn_detector.py`, a leak where the NPU handle (`self._rknn`) was not released on exception during `start()` was fixed. Individual `try-except` blocks were also added to `stop()` to guarantee NPU and camera resource release.

`sensor/button_handler.py`의 `_poll_loop`에서는 `chip`과 `line`을 `None`으로 초기화한 뒤 중첩 `try`로 감싸고, 초기화 실패 시 이미 열린 `chip`을 명시적으로 닫도록 재구성했습니다. `finally` 블록에서 None 체크 후 각 자원을 안전하게 해제합니다.\\
In `sensor/button_handler.py`'s `_poll_loop`, `chip` and `line` were initialized to `None` and wrapped in nested `try` blocks; on initialization failure, already-opened `chip` is explicitly closed. The `finally` block safely releases each resource after a None check.

`sensor/vl53l1x_hal.py`의 `start()` 예외 블록에서는 `self._tof`가 이미 열려 있는 경우 `close()`를 호출한 후 `None`으로 초기화하도록 수정했습니다.\\
In `sensor/vl53l1x_hal.py`'s `start()` exception block, if `self._tof` is already open, `close()` is now called before setting it to `None`.

`main.py`의 `write_row()` 호출은 `try-except`로 감싸 디스크 에러가 메인 루프 전체를 다운시키지 않도록 처리했습니다.\\
The `write_row()` call in `main.py` was wrapped in `try-except` to prevent disk errors from bringing down the entire main loop.

`tests/benchmark_rknn.py`에도 `try...finally` 구조를 적용하여 측정 도중 중단이 발생하더라도 NPU와 카메라 리소스가 확실히 정리되도록 개선했습니다.\\
`try...finally` was also applied to `tests/benchmark_rknn.py` so that NPU and camera resources are reliably cleaned up even on interrupted measurements.

---

### pytest 수집 문제 근본 해결\\Root-Cause Fix for pytest Collection Issues

`scripts/test_device.py`의 `TestResult` 클래스가 pytest 테스트 대상으로 오인되어 경고를 발생시키는 문제를 두 단계로 해결했습니다.\\
The issue where the `TestResult` class in `scripts/test_device.py` was mistaken by pytest as a test target was resolved in two steps.

초기에는 `__test__ = False` 속성을 추가해 클래스 단위로 수집을 차단했고, 이후 `pytest.ini`를 신규 추가하여 `testpaths = tests` 설정으로 pytest가 `tests/` 디렉터리만 탐색하도록 근본적으로 제한했습니다.\\
Initially, `__test__ = False` was added to block collection at the class level; then a new `pytest.ini` was added with `testpaths = tests` to fundamentally restrict pytest to scanning only the `tests/` directory.

또한 에이전트 역할 규칙(`.agents/AGENTS.md`)을 업데이트하여 에이전트가 프로젝트 코드를 직접 수정하지 않고 `feedback.txt`를 통해 코드 리뷰만 수행하는 '코드 리뷰어'로 역할을 명확히 한정했습니다.\\
The agent role rules in `.agents/AGENTS.md` were updated to clearly limit the agent's role to a "Code Reviewer" that only provides feedback via `feedback.txt` without directly modifying project code.

모든 수정 완료 후 pytest 72개 테스트가 전체 통과되었습니다.\\
After all fixes, all 72 pytest tests passed successfully.
