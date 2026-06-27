---
title: "RasEyes Phase 5: 시스템 최적화·안정화"
date: 2026-06-27 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [python, ema, latency, thermal, pwm, gpiod, numpy, alsa, opencv, systemd, pytest]
---

RasEyes Phase 5에서는 E2E 레이턴시 프로파일링, 발열 대응, 카메라 가림 감지, 배터리 경고 기능을 구현하고, 코드 리뷰 피드백 8개 항목을 반영했습니다.\\
In RasEyes Phase 5, system stability features were implemented including E2E latency profiling, thermal management, camera occlusion detection, and battery warning. Eight code review feedback items were also addressed.

---

### E2E 레이턴시 프로파일링 (Phase 5-1)\\E2E Latency Profiling (Phase 5-1)

`logs/logger.py`의 `FIELDNAMES`에 `latency_ms` 컬럼을 추가하고 `write_row()` 파라미터도 함께 확장했습니다.\\
The `latency_ms` column was added to `FIELDNAMES` in `logs/logger.py`, and the `write_row()` parameter was extended accordingly.

`main.py`에서는 `vision_ts` 기준으로 EMA(지수 이동 평균) 레이턴시를 측정하고, 400ms 초과 시 경고 로그와 CSV 기록을 남기도록 구현했습니다.\\
In `main.py`, EMA (Exponential Moving Average) latency is measured relative to `vision_ts`; when it exceeds 400ms, a warning log and a CSV entry are recorded.

---

### 발열 Graceful Degradation (Phase 5-2)\\Thermal Graceful Degradation (Phase 5-2)

`config.py`에 `THERMAL_THROTTLE_TEMP_C = 80.0`과 `THERMAL_THROTTLE_FPS = 5`를 추가했습니다.\\
`THERMAL_THROTTLE_TEMP_C = 80.0` and `THERMAL_THROTTLE_FPS = 5` were added to `config.py`.

`main.py`에는 `_thermal_event`를 추가하여 1초 주기로 온도를 체크하고, 임계 온도 초과 시 vision worker에 스로틀 슬립을 적용해 FPS를 낮춥니다.\\
In `main.py`, `_thermal_event` was added to check temperature every second; when the threshold is exceeded, a throttle sleep is applied to the vision worker to reduce FPS.

---

### 카메라 가림 감지 (Phase 5-3)\\Camera Occlusion Detection (Phase 5-3)

`config.py`에 `CAMERA_OCCLUSION_CHANGE_THRESH = 3.0`, `CAMERA_OCCLUSION_FRAMES = 15`, `CAMERA_OCCLUSION_COOLDOWN_SEC = 5.0` 세 파라미터를 추가했습니다.\\
Three parameters were added to `config.py`: `CAMERA_OCCLUSION_CHANGE_THRESH = 3.0`, `CAMERA_OCCLUSION_FRAMES = 15`, and `CAMERA_OCCLUSION_COOLDOWN_SEC = 5.0`.

`main.py`에서는 연속 프레임 간 `np.mean(|curr - prev|)`으로 픽셀 변화량을 측정합니다. 15프레임 연속으로 평균 변화량이 3.0 미만이면 카메라 가림으로 판단하여 HIGH 경보를 발령하고, 5초 쿨다운을 적용합니다.\\
In `main.py`, `np.mean(|curr - prev|)` measures pixel change between consecutive frames. If the average change stays below 3.0 for 15 consecutive frames, a HIGH alert is raised with a 5-second cooldown.

---

### 배터리 잔량 경고 (Phase 5-4)\\Battery Low Warning (Phase 5-4)

`config.py`에 `BATTERY_LOW_THRESHOLD_PCT = 20`, `BATTERY_CHECK_INTERVAL_SEC = 30.0`, `BATTERY_SYSFS_PATH`를 추가했습니다.\\
`BATTERY_LOW_THRESHOLD_PCT = 20`, `BATTERY_CHECK_INTERVAL_SEC = 30.0`, and `BATTERY_SYSFS_PATH` were added to `config.py`.

`main.py`에 `_read_battery_percent()` 함수를 신규 추가하여 30초 주기로 배터리 잔량을 확인하고, 20% 미만이 되면 MID 경보를 발령합니다.\\
A new `_read_battery_percent()` function was added to `main.py` to check battery level every 30 seconds, issuing a MID alert when it falls below 20%.
