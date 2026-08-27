---
title: "RasEyes: 보조배터리를 살리는 차등 기동, 그리고 한국어 TTS 삭제"
date: 2026-07-07 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [orangepi, piper, tts, onnxruntime, alsa, npu, python, embedded, troubleshooting]
---

전류 스파이크 완화 작업과, 공들여 살려놨던 한국어 TTS를 결국 영어로 갈아치웠다.\\
The current spike mitigation work and the Korean TTS, which had been carefully saved, were eventually replaced with English.

---

## 1. 부팅하자마자 다 같이 켜지면 보조배터리가 못 버틴다

RasEyes는 최종적으로 보조배터리 단독 구동이 목표다. 그런데 부팅 시퀀스를 보고 있으면 NPU, 카메라, 센서, 오디오가 거의 동시에 초기화된다. 각각의 기동 전류 스파이크가 한 시점에 겹치면 보조배터리의 과전류 보호(OCP)가 트립되면서 전원이 뚝 끊길 위험이 있었다.\\
RasEyes ultimately aims to run on a power bank alone. But watching the boot sequence, the NPU, camera, sensors, and audio all initialize almost simultaneously. If each component's inrush current spike overlaps at the same moment, the power bank's overcurrent protection (OCP) could trip and cut the power dead.

그래서 컴포넌트를 동시에 켜지 않고 순차적으로 기동하도록 바꿨다. `config.py`에 `STARTUP_STAGGER_SEC = 1.5` 상수를 두고, `RasEyesApp.start()`에서 vision → 1.5초 대기 → sensor → 1.5초 대기 → audio → 버튼 핸들러 순으로 하나씩 깨운다. Mock 모드에서는 지연 없이 즉시 기동하고, 실제 하드웨어(`use_hw=True`)일 때만 stagger를 적용하도록 했다.\\
So I changed the startup to bring components up sequentially instead of all at once. I added a `STARTUP_STAGGER_SEC = 1.5` constant to `config.py`, and in `RasEyesApp.start()` each component wakes up one by one: vision → wait 1.5s → sensor → wait 1.5s → audio → button handler. In Mock mode everything starts immediately with no delay; the stagger only applies on real hardware (`use_hw=True`).

한 가지 더 신경 쓴 부분은 부팅 안내음(TTS)이다. TTS 합성은 CPU를 꽤 먹는 작업이라, 이게 NPU 연속 추론 부하와 겹치면 또 전류가 튄다. 그래서 부팅 TTS 발화가 끝날 때까지 기다렸다가(`STARTUP_TTS_WAIT_TIMEOUT_SEC`) 다시 1.5초를 쉬고 나서야 vision/sensor 워커 스레드를 시작하도록 순서를 잡았다.\\
One more thing I paid attention to was the boot announcement (TTS). TTS synthesis is fairly CPU-heavy, and if it overlaps with continuous NPU inference the current spikes again. So the sequence now waits for the boot TTS utterance to finish (`STARTUP_TTS_WAIT_TIMEOUT_SEC`), rests another 1.5 seconds, and only then starts the vision/sensor worker threads.

---

## 2. 전류 스파이크는 부팅 때만 문제가 아니었다

부팅 순서만 손본다고 끝이 아니었다. 상시 운전 중에도 전류를 튀게 만드는 요인들이 있어서 같이 정리했다.\\
Fixing the boot order alone wasn't the end of it. There were factors causing current spikes during normal operation too, so I cleaned those up together.

먼저 Piper TTS가 쓰는 onnxruntime이 기본값으로 모든 CPU 코어를 끌어다 쓰고 있었다. `audio/piper_tts.py`에서 `onnxruntime.SessionOptions`를 몽키패치해 intra-op/inter-op 스레드를 각각 1개로 제한했다. 합성이 조금 느려지더라도 전류가 한 번에 튀는 것보다는 낫다.\\
First, onnxruntime used by Piper TTS was grabbing every CPU core by default. I monkey-patched `onnxruntime.SessionOptions` in `audio/piper_tts.py` to limit both intra-op and inter-op threads to one each. Even if synthesis gets a bit slower, that beats the current spiking all at once.

오디오 쪽도 손봤다. `JackAudioHAL`이 `sounddevice`(PortAudio)를 쓰고 있었는데, TTS와 서로 다른 경로로 ALSA를 잡으면서 충돌하는 문제가 있어 `aplay` subprocess 방식으로 전환해 TTS와 동일한 ALSA dmix 경로를 타게 했다. 그리고 이후 커밋에서는 재생할 때마다 ALSA 디바이스를 열고 닫는 대신 상주 스트림(`ResidentAudioStream`)을 유지하도록 바꿨다. 코덱과 앰프가 재생 때마다 껐다 켜지면서 생기는 전류 스파이크를 막기 위해서다. 이건 다시 까먹지 않게 프로젝트 규칙으로도 못 박아뒀다.\\
I also reworked the audio path. `JackAudioHAL` was using `sounddevice` (PortAudio), which grabbed ALSA through a different path than the TTS and caused conflicts, so I switched it to an `aplay` subprocess so it shares the same ALSA dmix path as the TTS. In a later commit, I also introduced a resident stream (`ResidentAudioStream`) that stays open instead of opening and closing the ALSA device on every playback — to prevent the current spikes caused by the codec and amp powering on and off repeatedly. I also nailed this down as a project rule so I don't forget it again.

이 조치들을 다 합친 상태로 보조배터리 단독 구동 테스트를 통과했다.\\
With all these measures combined, the power-bank-only operation test passed.

---

## 3. 한국어 TTS를 들어보니 너무 어색하다

실제 기기에서 합성된 한국어 음성을 들어봤는데, 발음과 억양이 너무 어색했다. 안내 음성으로 실사용하기엔 부적합하다고 판단했고, 고민 끝에 영어 TTS로 전환하기로 했다.\\
I finally listened to the synthesized Korean voice on the actual device — and the pronunciation and intonation were just too awkward. I judged it unfit for real use as a guidance voice, and after some deliberation decided to switch to an English TTS.

