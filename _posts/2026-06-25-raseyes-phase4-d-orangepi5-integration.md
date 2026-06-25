---
title: "RasEyes 개발일지: Orange Pi 5 통합, 센서 최적화"
date: 2026-06-25 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [orangepi5, vl53l1x, rknn, systemd, i2c, portaudio, sounddevice, tof, github-actions]
---

Orange Pi 5 환경에서 RasEyes 시스템의 통합 테스트와 센서 최적화를 성공적으로 완료했습니다.\\
I have successfully completed the integration testing and sensor optimization of the RasEyes system on the Orange Pi 5.

---

### 의존성 설치 및 테스트 자동화\\Dependency Setup and Test Automation

하드웨어 제어와 음향 출력을 위한 라이브러리(`sounddevice`, `gpiod`, `VL53L1X`) 설치를 완료했습니다.\\
I completed the installation of libraries (`sounddevice`, `gpiod`, `VL53L1X`) for hardware control and audio output.

단품 하드웨어 검증을 자동화하기 위해 `scripts/test_device.py`를 작성하여 CSICameraHAL과 VL53L1XHAL의 정상 동작을 빠르게 검증할 수 있도록 했습니다.\\
To automate hardware unit verification, I created `scripts/test_device.py` to quickly validate the operations of CSICameraHAL and VL53L1XHAL.

또한, 향후 NPU 모델 성능 측정을 위해 RKNN 추론 속도를 벤치마크할 수 있는 `scripts/bench_rknn.py` 스크립트도 함께 추가했습니다.\\
Additionally, I added the `scripts/bench_rknn.py` script to benchmark RKNN inference speeds for future NPU model performance measurements.

---

### ToF 센서 버그 수정 및 최적화\\ToF Sensor Bug Fixes and Optimization

`sensor/vl53l1x_hal.py` 모듈에서 라이브러리 참조 오류와 파라미터명 오류(`i2c_port`에서 `i2c_bus`로 수정)를 해결했습니다.\\
I resolved library reference and parameter name errors (changing `i2c_port` to `i2c_bus`) in the `sensor/vl53l1x_hal.py` module.

특히 ToF 센서의 모드를 LONG(3)에서 MEDIUM(2)으로 변경하여 성능과 안정성을 모두 확보했습니다.\\
Particularly, I changed the ToF sensor mode from LONG (3) to MEDIUM (2) to secure both performance and stability.

기존 LONG 모드는 최소 140ms의 타이밍 버짓이 필요한 상황에서 50ms로 무리하게 설정되어 데이터 갱신 속도가 약 1초까지 지연되고 만료 경고가 다량 발생했습니다.\\
The previous LONG mode, which required a minimum timing budget of 140ms, was aggressively set to 50ms, causing data update rates to delay up to ~1s and flooding expiration warnings.

MEDIUM 모드는 최대 3m 거리 측정과 33ms+ 버짓을 지원하여, 50ms 설정 하에서 약 10Hz의 안정적인 샘플링 속도를 보여주며 경고를 완전히 해결했습니다.\\
The MEDIUM mode supports up to a 3m range and a 33ms+ budget, which achieved a stable sampling rate of around 10Hz under the 50ms configuration, completely resolving the warnings.

---

### 예외 처리 강화 및 부팅 성능 향상\\Robust Exception Handling and Boot Performance

`main.py`에서는 `rknnlite2`나 `ultralytics`, `PortAudio` 등 특정 라이브러리가 미설치된 환경에서도 프로그램이 크래시되지 않고 정상적으로 대체 동작을 수행하도록 개선했습니다.\\
In `main.py`, I improved the program to execute a graceful fallback instead of crashing when specific libraries such as `rknnlite2`, `ultralytics`, or `PortAudio` are not installed.

특히 `_build_vision()` 함수 내부에서 NPU 변환 모델 파일(`.rknn`)의 존재 여부를 팩토리 단계에서 선제적으로 검사하여, 없을 경우 `MockVision`으로 안전하게 전환되도록 구현했습니다.\\
Specifically, I implemented a pre-check for the existence of the NPU conversion model file (`.rknn`) within the `_build_vision()` function at the factory stage, safely switching to `MockVision` if missing.

부팅 시 시스템이 자동 시작되도록 `systemd` 서비스 등록을 마쳤으며, 부팅 속도는 커널 3.8초, 유저스페이스 7.2초를 기록하여 총 11초 만에 완료되었습니다.\\
I completed the `systemd` service registration for auto-start on boot, achieving a total boot time of 11 seconds (kernel 3.8s + userspace 7.2s).

이는 하드웨어 통합 검증 KPI 기준인 45초를 큰 폭으로 단축한 수치입니다.\\
This significantly outperforms the hardware integration verification KPI target of 45 seconds.

음향 라이브러리인 `sounddevice`가 시스템에 설치된 `PortAudio`를 올바르게 감지하여 별도의 패키지 수동 설치 없이 `JackAudioHAL`이 매끄럽게 구동되는 점도 확인했습니다.\\
I also verified that the audio library `sounddevice` automatically detects the system's `PortAudio`, allowing `JackAudioHAL` to run smoothly without manual package installations.

---

### CI/CD 파이프라인 및 현재 운영 현황\\CI/CD Pipeline and Current Operational Status

GitHub Actions(ubuntu x86_64) 환경에서 PyTorch 모델을 RKNN 파일로 자동 변환하는 `.github/workflows/build_rknn.yml` 워크플로우를 구축했습니다.\\
I established a `.github/workflows/build_rknn.yml` workflow to automatically convert PyTorch models to RKNN files in a GitHub Actions (ubuntu x86_64) environment.

현재 Orange Pi 5 장비는 `MockVision` + `VL53L1XHAL` + `JackAudioHAL` 조합을 바탕으로 실측 거리 기반 경보 기능을 정상적으로 제공하고 있습니다.\\
Currently, the Orange Pi 5 device is successfully providing real-time distance-based alerts using the combination of `MockVision` + `VL53L1XHAL` + `JackAudioHAL`.

CSI 카메라는 `/dev/video11` V4L2 드라이버 한계로 인해 약 10 FPS를 기록하여 목표 수치인 15 FPS에 다소 못 미치는 성능을 보이지만, 향후 NPU 추론을 위해 `yolov8n.rknn` 파일만 배포하면 즉시 고속 NPU 추론 모드로 전환될 준비가 끝난 상태입니다.\\
Although the CSI camera records around 10 FPS due to `/dev/video11` V4L2 driver limitations—slightly below the 15 FPS target—it is fully prepared to transition immediately to high-speed NPU inference once the `yolov8n.rknn` file is deployed.
