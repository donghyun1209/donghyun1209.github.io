---
title: "RasEyes 개발일지 Orange Pi 5에 올리기"
date: 2026-06-21 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [orangepi, raseyes, hal, rknn, vl53l1x, systemd, python, embedded]
---

지금까지 맥북에서 Mock 환경으로만 돌리던 RasEyes를 실제 Orange Pi 5 하드웨어에서 구동하기 위한 HAL 구현체들을 전부 작성했다.\\
I finished writing all HAL implementations for running RasEyes on the actual Orange Pi 5 hardware, which had only been running in a Mock environment on MacBook until now.

코드 구조 자체는 건드리지 않고, 하드웨어별 구현체만 갈아끼우는 방식이다.\\
The code structure itself was not touched — only the hardware-specific implementations were swapped in.

---

## 1. 카메라 — CSI 카메라 HAL (Camera — CSI Camera HAL)

Orange Pi 5에 붙어 있는 OV13855 MIPI CSI 카메라를 OpenCV로 잡는 클래스를 만들었다.\\
A class was created to capture the OV13855 MIPI CSI camera attached to the Orange Pi 5 using OpenCV.

기존 `OpenCVCamera`랑 거의 똑같은데, 카메라 인덱스(0, 1...)가 아니라 장치 경로(`/dev/video11`)를 받는다는 점이 다르다.\\
It is almost identical to the existing `OpenCVCamera`, but the difference is that it takes a device path (`/dev/video11`) instead of a camera index (0, 1...).

버퍼 크기를 1로 설정해서 프레임 지연이 쌓이지 않게 했다. 해상도가 안 맞을 경우엔 소프트웨어 리사이즈로 폴백한다.\\
The buffer size was set to 1 to prevent frame latency from accumulating. If the resolution doesn't match, it falls back to software resize.

---

## 2. ToF 센서 — VL53L1X HAL (ToF Sensor — VL53L1X HAL)

VL53L1X 드라이버를 64비트 ARM(aarch64)에서 그냥 쓰면 세그폴트가 난다.\\
Using the VL53L1X driver as-is on 64-bit ARM (aarch64) causes a segfault.

ctypes 함수 포인터 크기가 틀려서 발생하는 버그로, 공식 이슈에도 올라온 known 버그다.\\
It's a known bug caused by incorrect ctypes function pointer sizes, already reported in the official issue tracker.

라이브러리 내부 C 함수 시그니처를 직접 재정의해서 패치했다.\\
The internal C function signatures in the library were manually redefined as a patch.

```python
lib = VL53L1X._TOF_LIBRARY
lib.initialise.restype = c_void_p
lib.getDistance.restype = c_uint16
# ... 나머지 argtypes도 명시
```

이거 안 하면 `start_ranging()` 호출 순간 프로세스가 죽는다.\\
Without this, the process dies the moment `start_ranging()` is called.

거리 읽기는 0mm(측정 범위 초과)이면 `TOF_OUT_OF_RANGE_CM`을 반환하고, 그 외엔 mm을 10으로 나눠서 cm로 돌려준다.\\
For distance reading, if the value is 0mm (out of measurement range), `TOF_OUT_OF_RANGE_CM` is returned; otherwise, mm is divided by 10 and returned as cm.

---

## 3. 오디오 — 이어폰 잭 HAL (Audio — Earphone Jack HAL)

`sounddevice` + `numpy`로 사인파를 만들어서 3.5mm 잭으로 출력한다.\\
A sine wave is generated with `sounddevice` + `numpy` and output through the 3.5mm jack.

HIGH(2000Hz), MID(1000Hz) 두 가지 주파수를 쓰고, 시작/끝 10ms에 페이드인·페이드아웃을 걸어서 "딱" 하는 클릭 노이즈를 없앴다.\\
Two frequencies are used — HIGH (2000Hz) and MID (1000Hz) — with a 10ms fade-in/fade-out at the start and end to eliminate the "click" noise.

비동기(`blocking=False`)라서 비프음이 나오는 동안에도 메인 루프가 멈추지 않는다.\\
Since it's asynchronous (`blocking=False`), the main loop doesn't stall while a beep is playing.

---

## 4. RKNN NPU 파이프라인 (RKNN NPU Pipeline)

Orange Pi 5에는 NPU가 달려 있다. PC에서 YOLOv8n 모델을 INT8로 변환해서(`scripts/export_rknn.py`) Orange Pi 5로 보내면, `RknnDetector`가 rknnlite2로 로드해서 추론한다.\\
The Orange Pi 5 has an NPU. The YOLOv8n model is converted to INT8 on a PC (`scripts/export_rknn.py`) and sent to the Orange Pi 5, where `RknnDetector` loads and runs inference via rknnlite2.

추론 흐름은 아래와 같다.\\
The inference flow is as follows.

> 640×640 리사이즈 → NPU 추론 → NMS 후처리 → 원본 해상도 bbox 역변환
>
> 640×640 resize → NPU inference → NMS post-processing → bbox inverse transform to original resolution

rknnlite2가 설치 안 된 환경이거나 모델 파일이 없을 때엔 자동으로 PyTorch CPU 추론으로 폴백되도록 factory 함수에서 처리해뒀다.\\
If rknnlite2 is not installed or the model file is missing, the factory function automatically falls back to PyTorch CPU inference.

---

## 5. 시스템 서비스 (System Service)

전원만 꽂으면 자동으로 실행되도록 systemd 서비스를 만들었다.\\
A systemd service was created so it runs automatically on power-on.

```ini
Environment="RASEYES_HW=1"
Restart=on-failure
RestartSec=5
```

`RASEYES_HW=1`을 환경변수로 넣어두면 `main.py`가 하드웨어 HAL을 선택한다.\\
With `RASEYES_HW=1` set as an environment variable, `main.py` selects the hardware HAL.

부팅 완료 시엔 MID → MID → HIGH 멜로디가 울려서 "준비됐다"는 신호를 준다. 시각장애인이 화면 없이 상태를 인지할 수 있게 하기 위해서다.\\
On boot completion, a MID → MID → HIGH melody plays as a "ready" signal — so visually impaired users can recognize the device state without a screen.

---

## 다음 할 일 (Next Steps)

- [ ] Orange Pi 5에서 의존성 설치 (`pip install -r requirements-rpi.txt`)
- [ ] 카메라 / ToF / 오디오 단품 테스트
- [ ] `RASEYES_HW=1 python main.py` 통합 테스트 (FPS≥15, 비프음 확인)
- [ ] `scripts/export_rknn.py`로 `yolov8n.rknn` 생성 후 scp로 전송
- [ ] RKNN 추론 속도 측정 (목표 < 60ms)
- [ ] systemd 서비스 등록 및 부팅 자동 시작 검증
