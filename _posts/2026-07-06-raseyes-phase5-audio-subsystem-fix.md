---
title: "RasEyes Phase 5: 오디오 서브시스템 안정화 & 카메라 ISP 버그 수정"
date: 2026-07-06 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [python, alsa, espeak-ng, tts, sounddevice, v4l2, media-ctl, isp, threading, portaudio, pytest, orange-pi]

---

### 1. BootSequence 누락 연동 수정\\BootSequence Missing Integration Fix

`audio/boot_sequence.py`는 이미 구현되어 있었지만 `main.py`에서 실제로 호출하는 코드가 없어 부팅 멜로디와 TTS가 실행되지 않는 문제가 있었습니다.\\
`audio/boot_sequence.py` was already implemented but never called from `main.py`, so the boot melody and TTS were silently skipped.

`main.py`에 `from audio.boot_sequence import BootSequence` import를 추가하고, `start()` 메서드 끝에 `BootSequence().play(self._audio, self._tts)` 호출을 추가하여 서비스 시작 시 부팅 시퀀스가 정상 실행되도록 했습니다.\\
`from audio.boot_sequence import BootSequence` was imported and `BootSequence().play(self._audio, self._tts)` was appended to the `start()` method, ensuring the boot sequence runs on every service start.

---

### 2. 카메라 가림 false positive 버그 수정\\Camera Occlusion False Positive Bug Fix

**증상**: 서비스 실행 직후부터 5초마다 카메라 가림 경보음(삑삑삑)이 계속 울렸습니다.\\
**Symptom**: The camera occlusion alarm (beep) triggered every 5 seconds immediately after service start.

**원인**: 재부팅 후 MIPI ISP 파이프라인이 초기화되지 않아 `/dev/video11`이 거의 검은 프레임(mean ≈ 1.8/255)을 출력했고, 프레임 변화량(delta ≈ 0.19)이 임계값(3.0) 미달로 15프레임 연속 감지되어 false positive가 발생했습니다. 정상 정적 장면에서도 센서 노이즈로 인한 delta ≈ 1.89로 임계값 3.0을 넘지 못해 false positive가 지속됐습니다.\\
**Root Cause**: After reboot, the MIPI ISP pipeline was uninitialised, causing `/dev/video11` to output near-black frames (mean ≈ 1.8/255). Frame delta (≈ 0.19) fell below the threshold (3.0) for 15 consecutive frames, triggering a false positive. Even in normal static scenes, sensor noise produced delta ≈ 1.89, still below 3.0.

**수정 내용**:\\
**Fix**:

`vision/csi_camera_hal.py`에 `_setup_isp_pipeline()` 메서드를 추가하여 `start()` 시 `media-ctl`과 `v4l2-ctl`로 ISP 파이프라인과 센서 노출을 자동 설정합니다.\\
`_setup_isp_pipeline()` was added to `vision/csi_camera_hal.py` to automatically configure the ISP pipeline and sensor exposure via `media-ctl` and `v4l2-ctl` on `start()`.

`config.py`에서 `CAMERA_OCCLUSION_CHANGE_THRESH`를 3.0에서 1.0으로 낮췄습니다. 정적 장면의 센서 노이즈(≈ 1.89)는 1.0보다 크고, 완전 차단 시의 delta(≈ 0.19)는 1.0 미만으로 두 케이스를 정확히 구분할 수 있습니다.\\
`CAMERA_OCCLUSION_CHANGE_THRESH` was lowered from 3.0 to 1.0 in `config.py`. Static-scene sensor noise (≈ 1.89) stays above 1.0 while full occlusion delta (≈ 0.19) stays below — cleanly separating the two cases.

```python
# config.py
CAMERA_OCCLUSION_CHANGE_THRESH = 1.0   # 변경 전 3.0 / was 3.0
CSI_SENSOR_SUBDEV  = "/dev/v4l-subdev1"
CSI_SENSOR_EXPOSURE = 20000
CSI_SENSOR_GAIN     = 200
```

---

### 3. espeak-ng 설치 및 TTS ALSA 충돌 수정\\espeak-ng Installation & TTS ALSA Conflict Fix

#### 3-1. espeak-ng 설치\\espeak-ng Installation

Orange Pi에 `espeak-ng`가 설치되어 있지 않아 `MockTts`로 fallback 되고 TTS가 무음 상태였습니다.\\
`espeak-ng` was not installed on the Orange Pi, causing silent fallback to `MockTts`.

```bash
sudo apt-get install -y espeak-ng   # 한국어 음성 'ko' 포함 / includes Korean voice 'ko'
```

#### 3-2. 부팅 멜로디 타이밍 버그\\Boot Melody Timing Bug

마지막 HIGH 비프(80ms, non-blocking) 재생이 완료되기 전에 TTS subprocess가 ALSA 장치를 선점해 멜로디 끝이 잘렸습니다.\\
The last HIGH beep (80ms, non-blocking) was being preempted by the TTS subprocess before playback finished, cutting off the melody's tail.

`audio/boot_sequence.py` 멜로디 루프 이후에 `time.sleep(AUDIO_BEEP_DURATION_MS / 1000 + 0.05)`를 추가하여 마지막 비프가 완전히 재생된 뒤 TTS가 시작되도록 했습니다.\\
`time.sleep(AUDIO_BEEP_DURATION_MS / 1000 + 0.05)` was added after the melody loop in `audio/boot_sequence.py` to ensure TTS starts only after the last beep completes.

#### 3-3. TTS ALSA 장치 충돌 (핵심 버그)\\TTS ALSA Device Conflict (Core Bug)

**원인**:
1. 기존 `EspeakTts`가 `subprocess.Popen(["espeak-ng", ...])` 방식으로 espeak-ng를 직접 ALSA `hw:2,0`에 접근하게 했지만, JackAudioHAL이 sounddevice로 이미 `hw:2,0`을 점유 중이었습니다 → `Device or resource busy`.\\
   The original `EspeakTts` called `subprocess.Popen(["espeak-ng", ...])`, making espeak-ng open ALSA `hw:2,0` directly — but JackAudioHAL already held `hw:2,0` via sounddevice → `Device or resource busy`.

2. sounddevice 방식으로 교체했으나 espeak-ng가 22050Hz mono WAV를 생성하는 반면 JackAudioHAL은 44100Hz stereo로 설정 → PortAudio 스트림 재구성 실패 (`Error opening OutputStream`).\\
   After switching to sounddevice, espeak-ng generated 22050Hz mono WAV while JackAudioHAL was configured at 44100Hz stereo → PortAudio stream re-configuration failed (`Error opening OutputStream`).

**수정**: `audio/tts.py`를 전면 재작성했습니다.\\
**Fix**: `audio/tts.py` was fully rewritten.

| 변경 전 / Before | 변경 후 / After |
|---------|---------|
| `subprocess.Popen(["espeak-ng", text])` — espeak-ng가 ALSA 직접 접근 | `subprocess.run(["espeak-ng", "--stdout", text])` — WAV 데이터만 생성 후 sounddevice로 단일 경로 재생 |
| 프로세스 기반 비동기 | `threading.Thread` 기반 비동기 |
| ALSA 장치 충돌 발생 | JackAudioHAL과 동일한 device_idx 공유 |
| 22050Hz mono 재생 | 44100Hz stereo로 업샘플링 후 재생 |

`main.py`의 `_build_tts()` 메서드에서 JackAudioHAL과 동일한 ES8388 `device_idx`를 `EspeakTts`에 전달하도록 수정했습니다.\\
In `main.py`, `_build_tts()` was updated to pass the same ES8388 `device_idx` used by JackAudioHAL to `EspeakTts`.

#### 3-4. 테스트 업데이트\\Test Update

`tests/test_tts.py`의 `TestEspeakTts`에서 `Popen` 기반 mock을 `_start_thread` / `_kill_current` mock으로 교체하여 새로운 스레드 기반 구조에 맞게 업데이트했습니다.\\
In `tests/test_tts.py`, the `TestEspeakTts` class was updated from `Popen`-based mocks to `_start_thread` / `_kill_current` mocks to match the new thread-based architecture.
