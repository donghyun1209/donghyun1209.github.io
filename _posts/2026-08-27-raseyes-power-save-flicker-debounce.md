---
title: "RasEyes: 절전 모드 깜빡임의 범인을 잡았다"
date: 2026-08-27 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [raseyes, tof, sensor, power-save-mode, debounce, hysteresis, scan-mode, embedded]
---

이 기기는 배터리로 돌아가고, 앞에 아무것도 없을 때는 카메라 속도를 늦춘다. 평소엔 초당 15장을 찍지만 정면이 비어 있으면 초당 4장으로 떨어뜨리고, 뭔가 가까이 나타나면 즉시 15장으로 되돌아간다.\\
This device runs on a battery, so when nothing is in front of it, it slows the camera down. It normally shoots 15 frames a second but drops to 4 when nothing's there, and jumps right back to 15 the moment something gets close.

여기까지는 처음부터 의도한 동작이다. 문제는 이 절전 모드가 1초에도 몇 번씩 켜졌다 꺼졌다를 반복하고 있었다는 것이다.\\
That part was intentional from the start. The problem was that this power-save mode was flipping on and off several times a second.

---

## 야외 로그를 보다가 뭔가 이상하다는 걸 느꼈다

야외 테스트 로그를 들여다보는데 눈에 걸리는 구간이 있었다.\\
While going through an outdoor test log, one stretch caught my eye.

```
16:41:21  물체 없음 → 절전 켜기 (초당 4장)
16:41:21  84cm에 물체! → 절전 끄기 (초당 15장)
16:41:22  물체 없음 → 절전 켜기
16:41:23  53cm에 물체! → 절전 끄기
16:41:24  물체 없음 → 절전 켜기
16:41:24  24cm에 물체! → 절전 끄기
```

숫자로 세어보니 11분 동안 51번 켜지고 50번 꺼졌다. 카메라가 자기 속도를 정하지 못하고 계속 흔들리고 있었던 것이다.\\
Counting it up, the mode had turned on 51 times and off 50 times in eleven minutes. The camera couldn't settle on a speed — it was just oscillating.

---

## 문턱은 있었는데, 아무 일도 안 하고 있었다

사실 이런 깜빡임을 막으려고 이미 문턱을 벌려둔 상태였다. 물체가 200cm보다 멀어지면 절전을 켜고, 150cm보다 가까워지면 끈다. 150~200cm 사이에서는 아무것도 하지 않는다. 에어컨이 26도에서 켜지고 24도에서 꺼지는 것과 같은 이유로, 경계선 근처에서 왔다갔다하지 말라고 일부러 벌려둔 구간이다.\\
There was already a threshold gap in place to prevent exactly this kind of flicker. Power-save turns on past 200cm and off under 150cm, and nothing happens in between. It's the same idea as an air conditioner that turns on at 26°C and off at 24°C — a deliberate gap so it doesn't chatter right at the boundary.

그런데 이 구간이 통째로 무용지물이었다. 거리 센서 값이 그 사이를 아예 지나가지 않고 있었다.\\
Except the gap wasn't doing anything at all. The distance sensor's readings simply never passed through it.

이유를 찾다 보니, 거리 센서가 "못 쟀다"는 상태를 400cm라는 숫자로 대신 내놓는다는 걸 알았다. 진짜로 400cm 앞에 뭔가 있다는 뜻이 아니라 "모름"이라는 표시였다. 그래서 실제로 들어오는 값은 트인 곳에서는 400, 뭔가 스치면 40 근처, 이렇게 400과 40 사이만 오갔다. 150~200이라는 벌려둔 구간을 매번 훌쩍 뛰어넘고 있었던 셈이다.\\
Digging into why, I found that the distance sensor reports 400cm whenever it fails to get a reading — not because something is actually 400cm away, but as a stand-in for "unknown." So the values it actually produced just bounced between 400 in open space and something like 40 whenever an object grazed by, jumping clean over the 150–200 gap every single time.

---

## 범인 하나가 증상 세 개를 만들고 있었다

이 하나의 깜빡임이 서로 달라 보이던 세 가지 문제를 동시에 만들고 있었다. 야외에서 반응이 평균 0.37초로 느려진 것, 11분에 19번씩 "카메라가 죽었다"는 오판이 뜬 것, 그리고 이틀째 원인을 못 찾고 있던 360도 둘러보기의 방향 누락까지.\\
This single flicker was quietly causing three problems that had looked unrelated: outdoor response time slowing to an average of 0.37 seconds, false "camera dead" alarms firing 19 times in eleven minutes, and — the one I'd been stuck on for two days — the 360-degree scan dropping directions.

셋 다 따로따로 원인을 찾고 있었는데, 알고 보니 뿌리가 하나였다.\\
I had been chasing three separate root causes, and it turned out there was only one.

---

## 절전 모드에 확인 절차 두 가지를 새로 넣었다

밝기나 카메라 속도를 판단할 때는 문턱값에다 "몇 번 연속으로 그래야 인정" 하는 두 번째 장치까지 걸어두고 있었는데, 절전 모드에는 이게 빠져 있었다. 오늘 그 빈틈을 채웠다.\\
For brightness and camera speed decisions, I already had a second safeguard on top of the threshold — requiring several consecutive confirmations before acting. Power-save mode was missing that. Today I filled that gap.

첫 번째는 연속 확인이다. 앞이 비었다는 판정이 8번 연속으로 나와야 절전에 들어가게 했다. 대략 0.5초 정도다.\\
The first fix is a consecutive-count check. The device now has to see "nothing in front" eight times in a row, roughly half a second, before it's allowed to enter power-save.

두 번째는 둘러보기와의 관계다. 360도 스캔 중에는 절전 모드에 아예 못 들어가게 막았고, 스캔을 시작하는 시점에 이미 절전 상태였다면 그 자리에서 즉시 빠져나오게 했다.\\
The second is how it interacts with the 360-degree scan. While a scan is running, the device can't enter power-save at all — and if it was already in power-save the moment the scan started, it gets forced out immediately.

### 들어갈 때만 까다롭게 하고, 나갈 때는 봐주지 않았다

나갈 때도 8번 연속을 요구하면 어떻게 될까 고민했다. 절전 중엔 프레임 간격이 0.25초라, 8번을 채우려면 물체가 코앞에 나타나도 2초나 지나야 반응한다는 계산이 나왔다. 장애물을 알려주는 기기가 2초씩 늦게 반응하는 건 용납할 수 없어서, 나가는 쪽은 문턱을 넘는 즉시 반응하도록 비대칭으로 만들었다.\\
I considered requiring eight consecutive frames to exit as well, but at power-save's 0.25-second frame interval, that would mean a two-second delay before the device reacted to something suddenly appearing right in front of it. That's not acceptable for a device whose whole job is warning about obstacles, so I made exit asymmetric — it reacts the instant the threshold is crossed, no consecutive check required.

### "막기"만으로는 부족했다

처음엔 둘러보기 중에 절전 진입을 막기만 하면 될 거라 생각했다. 그런데 문제가 됐던 첫 번째 사례가 하필 이미 절전에 들어가 있는 상태로 스캔이 시작된 경우였다. 진입을 막아봐야 이미 들어가 있는 상태는 손댈 수 없으니 소용이 없었다. 그래서 이미 들어가 있으면 즉시 빠져나오게 하는 처리까지 추가했다.\\
My first instinct was that blocking entry during the scan would be enough. But the case that had actually caused the bug was one where the scan started while the device was already inside power-save mode — blocking new entries does nothing for a state it's already in. So I added the forced-exit behavior on top of the block.
