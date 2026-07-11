---
title: "RasEyes 개발일지: Orange Pi 5 통합, 센서 최적화"
date: 2026-06-25 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [orangepi5, vl53l1x, rknn, systemd, i2c, portaudio, sounddevice, tof, github-actions]
---

의존성 설치를 마치고 드디어 보드에서 통합 테스트를 돌려봤다.\\
With the dependencies installed, I finally ran the integration test on the board.

---

## 1. 단품 테스트부터 자동화 (Automating the unit tests first)

`sounddevice`, `gpiod`, `VL53L1X` 라이브러리부터 깔았다. 매번 손으로 카메라 켜보고 센서 값 찍어보는 게 귀찮아서, `scripts/test_device.py`를 만들어 CSICameraHAL과 VL53L1XHAL이 제대로 동작하는지 한 번에 확인할 수 있게 했다. 나중에 NPU 성능도 재야 할 것 같아서 `scripts/bench_rknn.py`도 미리 만들어뒀다.\\
I installed the `sounddevice`, `gpiod`, and `VL53L1X` libraries first. Manually turning on the camera and checking sensor readings every time was tedious, so I wrote `scripts/test_device.py` to verify CSICameraHAL and VL53L1XHAL in one shot. Since I'd need to measure NPU performance eventually anyway, I also wrote `scripts/bench_rknn.py` ahead of time.

---

## 2. ToF 센서가 자꾸 헐떡거렸다 (The ToF sensor kept gasping for air)

`vl53l1x_hal.py`에서 파라미터명 오타(`i2c_port` → `i2c_bus`)부터 잡았다. 여기까진 쉬웠다.\\
First I fixed a parameter name typo in `vl53l1x_hal.py` (`i2c_port` → `i2c_bus`). That part was easy.

진짜 문제는 그다음이었다. 거리값이 갱신되는 속도가 점점 느려지더니 거의 1초에 한 번꼴로만 값이 바뀌었고, 로그에는 만료 경고가 쉴 새 없이 쌓였다. 처음엔 I2C 배선이나 전원 문제인 줄 알았다.\\
The real problem came right after. The distance reading's update rate kept slowing down until it was refreshing barely once a second, and the log was flooded with expiration warnings. At first I thought it was an I2C wiring or power issue.

원인은 타이밍 버짓이었다. 센서를 LONG 모드(최대 거리 측정용, 최소 140ms 타이밍 버짓 필요)로 설정해뒀는데, 실제 폴링 주기는 50ms로 짜여 있었다. 센서가 요구하는 시간보다 훨씬 짧은 주기로 계속 값을 요청하니 매번 "아직 준비 안 됐다"는 경고만 쌓이는 거였다.\\
The culprit was the timing budget. I had the sensor set to LONG mode (for maximum range, which needs at least a 140ms timing budget), but the actual polling loop ran every 50ms — far shorter than what the sensor needed. It kept getting asked for a value before it was ready, so warnings just piled up every cycle.

모드를 MEDIUM(최대 3m, 33ms+ 버짓)으로 낮췄다. 50ms 주기에서도 여유가 생겨서 안정적으로 약 10Hz 샘플링이 나왔고, 경고도 싹 사라졌다. 어차피 RasEyes가 감지해야 하는 장애물 거리는 3m면 충분했다.\\
I dropped the mode down to MEDIUM (max 3m, 33ms+ budget). That gave enough headroom at the 50ms cycle to get a stable ~10Hz sampling rate, and the warnings disappeared completely. 3m of range was plenty for the obstacles RasEyes actually needs to detect anyway.

---

## 3. 없는 라이브러리 때문에 죽는 걸 막았다 (Making sure missing libraries don't crash it)

`rknnlite2`나 `ultralytics`, `PortAudio`가 설치 안 된 환경에서 `main.py`가 그냥 죽어버리는 것도 손봤다. 특히 `_build_vision()`에서는 `.rknn` 모델 파일이 있는지부터 미리 검사해서, 없으면 `MockVision`으로 조용히 넘어가게 만들었다.\\
I also fixed `main.py` crashing outright in environments where `rknnlite2`, `ultralytics`, or `PortAudio` weren't installed. In `_build_vision()` especially, I now check upfront whether the `.rknn` model file exists, and if not, it quietly falls back to `MockVision`.

`systemd` 등록도 마쳤다. 부팅 시간을 재보니 커널 3.8초, 유저스페이스 7.2초로 총 11초. 목표였던 45초에 비하면 훨씬 여유 있는 수치라 만족스러웠다.\\
I finished the `systemd` registration too. Measured boot time came out to 11 seconds total (kernel 3.8s + userspace 7.2s) — comfortably under the 45-second target, which was satisfying.

`sounddevice`가 시스템에 깔린 `PortAudio`를 알아서 잘 찾아줘서, `JackAudioHAL`은 별다른 수동 설치 없이 바로 돌아갔다.\\
`sounddevice` automatically found the system's installed `PortAudio`, so `JackAudioHAL` worked right away with no manual setup needed.

---

## 4. GitHub Actions로 RKNN 빌드 자동화 (Automating the RKNN build with GitHub Actions)

PyTorch 모델을 RKNN 파일로 변환하는 작업을 매번 로컬에서 하기 귀찮아서, GitHub Actions에 `build_rknn.yml` 워크플로우를 만들어뒀다.\\
Doing the PyTorch-to-RKNN model conversion locally every time was a pain, so I set up a `build_rknn.yml` GitHub Actions workflow. 

---

## 5. 요약 및 교훈 (Summary & lessons)

지금 Orange Pi 5는 `MockVision` + `VL53L1XHAL` + `JackAudioHAL` 조합으로 거리 기반 경보는 정상 동작 중이다. CSI 카메라는 `/dev/video11` 드라이버 한계로 목표였던 15 FPS에 못 미치는 10 FPS 정도가 나오는데, `yolov8n.rknn`만 배포하면 바로 NPU 추론으로 갈아탈 준비는 끝나 있다.\\
Right now the Orange Pi 5 is running distance-based alerts fine with the `MockVision` + `VL53L1XHAL` + `JackAudioHAL` combo. The CSI camera runs around 10 FPS due to `/dev/video11` driver limits, short of the 15 FPS target, but it's fully ready to switch over to NPU inference the moment `yolov8n.rknn` gets deployed.

오늘 배운 건, 경고 로그가 쌓인다고 무조건 배선이나 전원을 의심할 게 아니라 설정값과 실제 주기가 서로 맞는지부터 확인해야 한다는 거였다. 스펙 문서를 안 읽고 감으로 값을 넣었다가 한참 헤맸다.\\
Today's lesson: when warnings pile up in the logs, don't jump straight to blaming wiring or power — check whether the configured value actually matches the real polling cycle first. I set a value by gut feeling instead of reading the spec sheet, and it cost me a good chunk of time.
