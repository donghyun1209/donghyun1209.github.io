---
title: "RasEyes 프로젝트를 마무리하며"
date: 2026-08-31 09:00:00 +0900
categories: [RasEyes, 회고]
tags: [raseyes, retrospective, embedded, npu, tof, camera, navigation, testing, wearable]
---

지난 5월 29일 오렌지파이 5에 우분투를 올리는 것부터 시작해서, 오늘로 석 달이다. 그 사이에 만들어진 포스트가 열두 개다. 여기서 한 번 끊고 정리하고 넘어가려 한다.\\
It's been three months since May 29, when I started by just flashing Ubuntu onto an Orange Pi 5. Twelve posts came out of that stretch. Here I want to draw a line and take stock before moving on.

---

## 처음 세운 목표는 이거였다

지팡이는 바닥 장애물만 안다. 가슴·머리 높이에 있는 간판, 나뭇가지, 트럭 적재함 같은 건 지팡이로 못 잡는다. 카메라 비전 AI와 ToF 거리 센서를 체스트 스트랩에 묶어서, 두 손은 자유롭게 둔 채 그 사각지대를 이어폰 소리로 알려주자는 게 출발점이었다.\\
A cane only knows about obstacles on the ground. Things at chest and head height — signboards, branches, the back of a truck — are invisible to it. The starting point was strapping a camera vision AI and a ToF distance sensor to the chest, keeping both hands free, and covering that blind spot with sound through an earphone.

---

## 그동안 있었던 일을 한 줄씩 정리하면

**2026-05-29 — 첫 삽질**: 오렌지파이 5에 우분투를 올리는 것부터가 이미 트러블슈팅이었다.\\
Just getting Ubuntu onto the Orange Pi 5 was already a troubleshooting exercise.

**2026-06-21 — 맥북에서 실제 보드로**: 카메라는 순조로웠지만, ToF 센서는 에러 메시지 하나 없이 프로세스를 죽였다. 원인은 aarch64에서만 터지는 라이브러리 버그였다.\\
The camera came over smoothly, but the ToF sensor killed the process with no error at all — a library bug that only surfaces on aarch64.

**2026-06-26 — NPU에 모델 올리기**: YOLOv8n을 INT8로 양자화해서 드디어 NPU 위에서 돌렸다.\\
Quantized YOLOv8n to INT8 and finally got it running on the NPU.

**2026-07-07 — 보조배터리와의 싸움**: 부품들이 한꺼번에 켜지면서 보조배터리 과전류 보호가 트립될 뻔했다. 차등 기동으로 막았고, 그 김에 어색했던 한국어 TTS는 영어로 갈아치웠다.\\
Every component powering on at once nearly tripped the power bank's overcurrent protection. Fixed it with staggered startup, and replaced the awkward Korean TTS with English along the way.

**2026-07-09 — 처음으로 하나가 됐다**: 방열판 달린 오렌지파이에 카메라와 이어폰까지 붙은, 착용 가능한 형태가 처음 나왔다.\\
The first wearable, assembled form — Orange Pi with heatsink, camera, and earphone all wired together.

**2026-07-29 — 보기 시작하니 안 보이던 문제가 보였다**: 경보 전후 사진과 상태 로그를 남기게 했더니, 카메라가 거꾸로 꽂혀 있었고 노출도 고정된 채였다는 게 곧바로 드러났다.\\
Started saving footage and telemetry around every alert, and it immediately exposed that the camera was mounted upside down and its exposure was frozen.

**2026-07-30 — 카메라가 스스로 눈을 뜨게 만들었다**: 자동 밝기 조절을 직접 짜고, 거리 센서의 가짜 평균 내기도 함께 고쳤다.\\
Built auto-exposure myself, and fixed the distance sensor's meaningless "average of the same value three times" along the way.

**2026-08-06 — 방향을 바꿨다**: 벽 옆을 걸으면 무조건 울리는 게 버그가 아니라 설계대로였다는 걸 인정하면서, "지팡이를 대체"에서 "지팡이보다 먼저, 지팡이가 못 보는 높이까지 본다"로 목표 자체를 바꿨다.\\
Admitted that ringing every time I walked past a wall wasn't a bug but the design working as intended, and changed the mission itself — from "replace the cane" to "see further ahead, and higher, than the cane ever could."

**2026-08-10 — 흩어진 데이터를 한 화면에**: 로그와 사진이 따로 놀아서 뭐가 문제였는지 알 방법이 없었다. 달력과 KPI 카드, 동기화된 그래프로 묶은 뷰어를 만들었다.\\
Logs and photos lived in separate places with no way to see what actually went wrong — built a viewer that ties them together with a calendar, KPI cards, and synced charts.

**2026-08-11 — 부품 없이 스캔 기능을 열었다**: 새 버튼을 사는 대신 이미 달려 있는 전원 버튼을 `grab()`으로 가로채서 360도 스캔 트리거로 재활용했다.\\
Instead of buying a new button, grabbed the existing power button with `grab()` and repurposed it as the 360-degree scan trigger.

**2026-08-27 — 깜빡임 하나가 세 가지 증상이었다**: 절전 모드가 1초에 몇 번씩 켜졌다 꺼지면서, 반응 지연·오탐지·둘러보기 오류를 동시에 만들고 있었다. 디바운스를 넣어서 한 번에 잡았다.\\
One flickering power-save mode was quietly causing three different symptoms at once — slow reactions, false "camera dead" readings, and a scan bug. One debounce fix caught all three.

**2026-08-30 — 길을 알려주기 시작했다**: 목적지 안내를 붙이자마자, 모르는 지시를 "직진하세요"로 잘못 말하는 버그와 장애물 경보가 길안내에 묻히는 버그를 발견했다. 테스트가 하나도 없었다는 게 근본 원인이었고, 그 자리에서 28개를 새로 붙였다.\\
The moment navigation went in, I found a bug that mistranslated an unknown instruction into "proceed straight" and another that let obstacle warnings get swallowed by navigation speech — both traced back to having zero automated tests for that logic, so I added 28 on the spot.

---

## 지금 와서 보면

3개월을 관통하는 패턴이 하나 있다. 문제를 고친 게 아니라, **문제를 볼 수 있게 만든 다음에야** 고칠 수 있었다는 것이다. 관찰 장치(7/29), 로그 뷰어(8/10), 테스트(8/30) — 이 세 개는 전부 새 기능이 아니라 "지금 뭐가 벌어지고 있는지 보이게 하는" 작업이었고, 그 직후마다 몰랐던 문제가 튀어나왔다.\\
One pattern runs through all three months: I couldn't fix a problem until I could first *see* it. Telemetry (7/29), the log viewer (8/10), tests (8/30) — none of those three were new features. Each was just "make what's actually happening visible," and each time, a problem I hadn't known about immediately fell out.

가장 크게 바뀐 건 8월 6일이다. "지팡이를 대체하겠다"는 목표를 붙잡고 있는 동안에는 벽 옆을 지날 때마다 울리는 게 버그처럼 느껴졌다. 목표를 "지팡이보다 먼저, 지팡이가 못 보는 곳까지"로 바꾸고 나서야 그게 당연한 동작이라는 걸 인정할 수 있었다. 목표를 잘못 잡으면 잘 만든 것도 실패로 보인다는 걸 몸으로 배운 하루였다.\\
The biggest shift was August 6th. As long as I was holding onto "replace the cane," ringing every time I passed a wall felt like a bug. Only after changing the goal to "see further and higher than the cane can" could I admit that behavior was correct all along. I learned first-hand that the wrong goal can make something well-built look like a failure.

지금 기기는 처음 문제 정의였던 "상단 사각지대 감지"에 더해, 버튼 하나로 주변을 훑는 360도 스캔과 목적지까지 안내하는 길찾기까지 붙어 있다. 테스트는 365개다. 물론 아직 다 끝난 건 아니다. 실외에서 더 오래, 더 다양한 환경에서 걸어봐야 하고, 방향 추정이 시간만으로 충분한지도 검증이 더 필요하다. 하지만 처음 만들었던 관찰 장치와 뷰어와 테스트가 이제는 그 다음 문제를 스스로 찾아줄 것이다.\\
Right now the device covers the original problem — top-blind-spot detection — plus a one-button 360-degree scan and turn-by-turn navigation to a destination. The test suite sits at 365. It's not finished, of course — it still needs longer walks across more varied environments, and whether time-based direction estimation is precise enough still needs proving. But the telemetry, the viewer, and the tests I built along the way will be the ones surfacing whatever comes next.

여기서 일단 마침표를 찍는다.\\
This is where I'm putting the period, for now.
