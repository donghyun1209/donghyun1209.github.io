---
title: "RasEyes: 버튼을 새로 안 달고 스캔 모드를 켰다"
date: 2026-08-11 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [raseyes, linux, evdev, input-device, power-button, tts, orangepi, embedded]
---

지난번에 정한 새 기능은 "버튼을 누르고 한 바퀴 돌면 주변에 뭐가 있는지 말해주는" 360도 스캔 모드였다. 오늘부터 그걸 만들기 시작했다.\\
The new feature I settled on last time was a 360-degree scan mode: press a button, turn around once, and the device tells you what's around you. Today I started building it.

이 기능은 다섯 조각으로 나뉜다. 버튼을 눌렀다는 신호를 받는 것, 스캔의 시작과 끝을 관리하는 것, 도는 동안 카메라와 거리센서를 같은 시점으로 묶어 찍는 것, 같은 물건을 두 번 세지 않는 것, 그리고 "정면에 의자 셋" 같은 문장으로 말하는 것.\\
It breaks down into five pieces: catching the signal that the button was pressed, managing when a scan starts and ends, capturing the camera and distance sensor at the same instant while turning, not counting the same object twice, and finally saying something like "three chairs in front of you."

오늘은 이 중 첫 번째 하나만 붙잡았다. 그런데 이게 예상 밖으로 재미있었다.\\
Today I only worked on the first one. And it turned out to be far more interesting than I expected.

---

## 1. 부품을 사는 대신 이미 달려 있는 버튼을 봤다

물리 버튼이 하나 필요했다. 처음엔 당연히 외부 스위치를 사서 GPIO 핀에 연결하는 걸 생각했고, 실제로 배선까지 다 그려놨다.\\
I needed one physical button. My first instinct was obviously to buy an external switch and wire it to a GPIO pin — I had even finished drawing out the wiring.

그런데 잠깐 멈춰서 보니, 이 보드엔 이미 버튼이 하나 달려 있었다. 전원 버튼이다.\\
But then I stopped for a second and realized this board already has a button on it. The power button.

굳이 부품을 새로 사서 배선하는 대신, 이미 있는 전원 버튼을 스캔 트리거로 재활용하면 안 되나? 오늘 작업은 이 질문에서 시작됐다.\\
Instead of buying a new part and wiring it up, why not just repurpose the power button that's already there as the scan trigger? That question is where today's work started.

문제는 하나였다. 전원 버튼은 이름 그대로 누르면 전원을 끄는 버튼이다. 이걸 다른 용도로 쓰려면, 소프트웨어가 시스템의 기본 종료 동작보다 먼저 이 버튼을 가로챌 수 있어야 했다.\\
There was one problem. A power button is, by definition, the button that turns the power off. To use it for anything else, my software would have to intercept it before the system's default shutdown behavior got to it.

---

## 2. 전원 버튼은 입력장치였다

확인해보니 반가운 사실이 나왔다. 이 전원 버튼은 그냥 하드웨어 배선이 아니라, 리눅스 커널이 `rk805 pwrkey`라는 이름의 정식 입력장치(`/dev/input/event0`)로 인식하고 있었다.\\
Checking it out gave me good news. This power button isn't just raw hardware wiring — the Linux kernel recognizes it as a proper input device named `rk805 pwrkey`, sitting at `/dev/input/event0`.

키보드나 마우스처럼 이벤트를 읽을 수 있는 표준 장치라는 뜻이다. 파이썬의 `evdev` 라이브러리로 이 장치를 열면 버튼을 누르고 뗄 때마다 `KEY_POWER` 이벤트가 그대로 들어오고, 눌린 시간까지 밀리초 단위로 잴 수 있다.\\
That means it's a standard device whose events I can read, just like a keyboard or a mouse. Opening it with Python's `evdev` library gives me a `KEY_POWER` event every time the button goes down and comes back up — and I can measure how long it was held, down to the millisecond.

<!-- NEED: rk805 pwrkey가 /dev/input/event0로 잡힌 장치 목록 출력 화면 -->

---

## 3. 문제는 읽는 게 아니라 가로채는 것이었다

이벤트를 읽는 것 자체는 쉬웠다. 진짜 문제는 평소에 이 버튼을 누르면 시스템이 즉시 전원을 끈다는 점이었다.\\
Reading the events was the easy part. The real problem was that pressing this button normally shuts the system down immediately.

우리 코드가 이벤트를 조용히 구경만 하고 있으면, 버튼을 누르는 순간 시스템의 기본 종료 로직이 먼저 반응해서 기기가 그냥 꺼져버린다.\\
If my code just sits there quietly watching the events, the system's default shutdown logic wins the race the moment the button is pressed, and the device simply powers off.

해결책은 리눅스 입력장치의 `grab()`이었다. 이 장치를 우리 프로세스가 배타적으로 점유하면, 다른 어떤 프로그램도 — 시스템의 종료 담당 프로세스까지 포함해서 — 이 버튼의 이벤트를 못 받는다. 우리만 받는다.\\
The answer was `grab()` on the Linux input device. If my process takes exclusive ownership of the device, no other program — including whatever process handles shutdown — receives this button's events at all. Only I do.

```python
device.grab()  # 이 순간부터 전원버튼은 오직 이 프로세스만 본다
```

이게 마음에 드는 이유가 하나 더 있다. 우리 서비스가 꺼지거나 죽으면 grab이 자동으로 풀린다는 것이다. 그러면 버튼은 즉시 원래의 "누르면 종료" 동작으로 돌아간다.\\
There's another reason I like this. If my service stops or crashes, the grab is released automatically, and the button instantly goes back to its original "press to shut down" behavior.

즉, 스캔 기능이 살아있을 땐 버튼이 스캔 트리거로 동작하고, 서비스가 없을 땐 그냥 원래 전원 버튼으로 돌아간다. 별도의 안전장치를 만들 필요 없이 자연스럽게 안전한 구조다.\\
So while the scan feature is alive the button acts as a scan trigger, and when the service isn't running it's just a power button again. It's safe by construction, without me having to build any fallback for it.

실제로 오렌지파이에 올려서 테스트했다. grab이 걸린 상태에서 버튼을 눌러도 기기는 안 꺼졌고, 대신 스피커에서 안내 음성이 나왔다.\\
I actually put it on the Orange Pi and tested it. With the grab in place, pressing the button didn't power anything off — the speaker played the announcement instead.

<!-- NEED: grab 상태에서 버튼을 눌러 스캔 안내 음성이 나오는 테스트 장면 -->

---

## 4. "Scanning"의 S가 통째로 안 들렸다

버튼을 누르면 TTS로 "Scanning mode..."라고 안내하게 만들었는데, 들어보니 자꾸 "canning mode"로 들렸다. 맨 앞의 S가 통째로 빠지는 것이다.\\
I made it announce "Scanning mode..." through TTS when the button is pressed, but what I actually heard was "canning mode" every time. The leading S was gone entirely.

몇 번을 다시 눌러봐도 매번 똑같았다. 반면 기존에 쓰던 "Danger!" 같은 경고 문구에서는 이런 일이 없었다.\\
No matter how many times I pressed it, it came out the same. Meanwhile the existing warning phrases like "Danger!" never had this problem.

패턴이 하나 보였다. S처럼 약하게 시작하는 자음은 이 TTS 모델이 문장 맨 앞에서 자주 삼키고, D처럼 강하게 튀는 자음은 안 그런다는 것이다.\\
A pattern showed up: this TTS model tends to swallow soft-onset consonants like S at the very start of a sentence, while hard-attack consonants like D come through fine.

재생 버퍼 코드를 다시 봐도 소프트웨어가 앞부분을 잘라낼 이유는 없었으니, 결론은 발음 자체의 문제였다.\\
I went back through the playback buffer code and found no reason software would be clipping the front, so the conclusion was that it was a pronunciation problem, not a code one.

고치는 법은 허무할 만큼 간단했다. 문장을 "Scanning..."이 아니라 "Please turn around. Scanning..."으로 바꿔서 강한 자음으로 시작하게 만든 것이다. 전달하는 정보는 완전히 같고, 안 씹히는 순서로만 바꿨다.\\
The fix was almost anticlimactically simple: change the line from "Scanning..." to "Please turn around. Scanning...", so it begins on a hard consonant. Exactly the same information, just reordered so nothing gets eaten.

