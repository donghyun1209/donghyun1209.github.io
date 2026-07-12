---
title: "RasEyes 개발일지: Orange Pi 5에 올리기"
date: 2026-06-21 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [orangepi, raseyes, hal, rknn, vl53l1x, systemd, python, embedded]
---

지금까지 RasEyes는 맥북 위에서만 돌아가는 물건이었습니다. 이제 진짜 Orange Pi 5 보드에 진짜 카메라와 진짜 센서를 붙여볼 차례가 됐습니다.\\
Until now, RasEyes had only ever run on my MacBook. It was finally time to hook it up to a real Orange Pi 5 board with real hardware.

다행히 코드 구조는 손대지 않고 하드웨어별 구현체(HAL)만 갈아끼우면 되도록 짜뒀습니다. 문제는 그 "갈아끼우는" 과정에서 하나씩 사고가 터졌다는 것입니다.\\
Luckily I'd built the code so only the hardware-specific implementations (HAL) needed swapping — the structure itself wouldn't change. The problem was that things broke one by one during that swap.

---

## 1. 카메라는 그럭저럭 순조로웠다 (The camera went smoothly enough)

Orange Pi 5에 붙어 있는 OV13855 MIPI CSI 카메라를 OpenCV로 잡는 것부터 시작했습니다. 기존에 쓰던 `OpenCVCamera` 클래스를 거의 그대로 재활용했고, 카메라 인덱스(0, 1...) 대신 장치 경로(`/dev/video11`)를 받도록 한 줄만 바꿨습니다.\\
I started by capturing the OV13855 MIPI CSI camera on the Orange Pi 5 with OpenCV. I reused the existing `OpenCVCamera` class almost as-is, just changing it to take a device path (`/dev/video11`) instead of a camera index (0, 1...).

버퍼 크기를 1로 안 잡으면 프레임이 계속 밀려서 화면이 과거를 보여주는 현상이 생긴다는 걸 다른 프로젝트에서 겪어본 적이 있어서, 이번엔 미리 버퍼 크기를 1로 박아뒀습니다. 해상도가 안 맞을 때는 소프트웨어 리사이즈로 폴백하게만 해두고 넘어갔습니다.\\
I already knew from a past project that skipping a buffer size of 1 makes frames pile up and the feed lag behind reality, so I set the buffer size to 1 from the start this time. I just added a software-resize fallback for resolution mismatches and moved on.

---

## 2. ToF 센서가 다짜고짜 프로세스를 죽였다 (The ToF sensor just killed the process)

문제는 VL53L1X 거리 센서였습니다. 라이브러리를 붙이고 `start_ranging()`을 호출한 순간, 에러 메시지 한 줄 없이 파이썬 프로세스가 그냥 죽어버렸습니다.\\
The real problem was the VL53L1X distance sensor. The moment I wired up the library and called `start_ranging()`, the Python process just died — no error message, nothing.

배선을 다시 확인해봐도 멀쩡했고, I2C 주소도 정상으로 잡혔습니다. 검색을 좀 해보니 64비트 ARM(aarch64)에서 이 드라이버를 그대로 쓰면 세그폴트가 난다는 게 공식 이슈 트래커에 이미 올라와 있는 known 버그였습니다.\\
I double-checked the wiring — fine. The I2C address was picked up correctly too. A bit of searching turned up a known bug already reported on the official issue tracker: using this driver as-is on 64-bit ARM (aarch64) causes a segfault.

원인은 ctypes 함수 포인터 크기였습니다. 라이브러리 내부 C 함수 시그니처를 직접 재정의해서 패치하는 수밖에 없었습니다.\\
The cause was a ctypes function pointer size mismatch. The only fix was to manually redefine the internal C function signatures myself.

```python
lib = VL53L1X._TOF_LIBRARY
lib.initialise.restype = c_void_p
lib.getDistance.restype = c_uint16
# ... 나머지 argtypes도 명시
```

이 몇 줄을 안 넣으면 `start_ranging()`을 부르는 순간 그대로 프로세스가 죽습니다. 패치하고 나니 거리 읽기는 멀쩡하게 동작했습니다. 0mm(측정 범위 초과)면 `TOF_OUT_OF_RANGE_CM`을 반환하고, 그 외엔 mm을 10으로 나눠 cm로 돌려주게 정리했습니다.\\
Without these few lines, the process dies the instant `start_ranging()` is called. Once patched, distance reading worked fine — 0mm (out of range) returns `TOF_OUT_OF_RANGE_CM`, otherwise mm is divided by 10 and returned as cm.

---

## 3. 오디오랑 RKNN 파이프라인은 미리 만들어만 뒀다 (Audio and the RKNN pipeline — built but untested)

이어폰 잭 출력은 `sounddevice`와 `numpy`로 사인파를 만들어서 내보내는 방식으로 짰습니다. HIGH(2000Hz)와 MID(1000Hz) 두 톤을 쓰고, 시작·끝 10ms에 페이드인·아웃을 걸어서 "딱" 하는 클릭 노이즈를 없앴습니다. 비동기로 재생되기 때문에 비프음이 나오는 동안에도 메인 루프는 멈추지 않습니다.\\
For earphone output, I generate a sine wave with `sounddevice` + `numpy`. I use two tones — HIGH (2000Hz) and MID (1000Hz) — with a 10ms fade-in/out at the start and end to kill the "click" noise. Playback is asynchronous, so the main loop never stalls while a beep plays.

NPU 추론 쪽은 PC에서 YOLOv8n을 INT8로 변환해서(`scripts/export_rknn.py`) 보드로 보내면 `RknnDetector`가 rknnlite2로 로드해 추론하는 구조로 짜뒀습니다. rknnlite2가 없거나 모델 파일이 없으면 자동으로 PyTorch CPU 추론으로 떨어지게 factory 함수에 폴백을 넣어뒀습니다. 아직 실제 보드에서 돌려본 적은 없습니다.\\
For NPU inference, I convert YOLOv8n to INT8 on my PC (`scripts/export_rknn.py`), send it to the board, and `RknnDetector` loads and runs it via rknnlite2. If rknnlite2 or the model file is missing, the factory function falls back to PyTorch CPU inference automatically. I haven't actually run this on the board yet.

---

## 4. 전원만 꽂으면 알아서 켜지게 (Power on, and it just starts)

`systemd` 서비스로 등록해서 전원만 꽂으면 자동으로 실행되게 만들었습니다. `RASEYES_HW=1` 환경변수를 넣어두면 `main.py`가 하드웨어 HAL을 골라 쓰는 식입니다.\\
I registered it as a `systemd` service so it starts automatically on power-on. Setting the `RASEYES_HW=1` environment variable makes `main.py` pick the hardware HAL.

부팅이 끝나면 MID → MID → HIGH 멜로디가 울리게 해뒀습니다. 화면 없이도 "준비됐다"는 걸 알 수 있어야 하니까요.\\
On boot completion, it plays a MID → MID → HIGH melody — so it's clear the device is ready even without a screen.

---

## 5. 요약 및 교훈 (Summary & lessons)

오늘 가장 크게 배운 건, "공식 라이브러리도 내가 쓰는 플랫폼에서는 그냥 안 돌아갈 수 있다"는 것입니다. VL53L1X 세그폴트는 배선도 아니고 제 코드도 아니고 라이브러리의 aarch64 미지원 버그였습니다. 에러 메시지 없이 죽는 증상을 만나면 일단 아키텍처부터 의심해봐야 한다는 걸 배웠습니다.\\
The biggest lesson today: even an official library can just not work on the platform you're actually using. The VL53L1X segfault wasn't the wiring or my code — it was the library's own aarch64 support bug. When something dies with no error message at all, architecture mismatch should be one of the first things to suspect.

이제 실제 보드에 의존성을 깔고, 카메라·ToF·오디오를 하나씩 단품 테스트해볼 차례입니다.\\
Next up: installing the dependencies on the actual board and testing the camera, ToF, and audio one by one.
