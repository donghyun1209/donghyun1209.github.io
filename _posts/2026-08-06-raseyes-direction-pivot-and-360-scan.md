---
title: "RasEyes: 방향을 바꿨다"
date: 2026-08-06 09:00:00 +0900
categories: [RasEyes, 회고]
tags: [raseyes, sensor, tof, camera, imu, ux, wayfinding, embedded]
---

오늘은 코드를 거의 안 고쳤다. 대신 이 기기가 무엇을 하는 물건인지를 바꿨다. 하루 종일 한 일이 그거다.\\
Today I barely touched the code. Instead, I changed what this device is even supposed to be. That's what I spent the whole day on.

밖에서 25분 걷고 데이터를 뽑아봤는데, 숫자 하나가 눈에 걸렸다.\\
I walked outside for 25 minutes and pulled the data, and one number caught my eye.

| 항목 | 결과 | 목표 |
|---|---|---|
| 속도 | 초당 13.8장 | ✅ |
| 온도 | 최대 49.9도 | ✅ |
| 반응 시간 | 0.2초 | ✅ |
| 경고 횟수 | 1분에 10번 | ❌ (1분에 1번) |

성능은 다 통과했다. 10배 시끄러운 것만 빼고.\\
Every performance number passed. Except for being ten times noisier than it should be.

1분에 10번이면 6초마다 한 번이다. 실제로 착용해보면 안다. 그냥 계속 떠든다.\\
Ten times a minute means once every six seconds. You only really understand it once you wear the thing — it just never shuts up.

---

## 1. 벽 옆을 걸으면 무조건 울렸다

경고 사진 30장에 저장된 기록을 하나씩 대조해봤다.\\
I went through the 30 saved warning photos one by one and compared them against the logs.

20장, 그러니까 67%가 카메라가 아무것도 못 찾은 상태에서 울린 것이었다.\\
20 of them — 67% — had gone off while the camera hadn't recognized anything at all.

무슨 일이 벌어지고 있었냐면, 거리 센서가 1미터 안에 뭔가 있다고 알리면 카메라는 그게 뭔지 못 알아보고, 기기는 모르겠지만 가까우니까 일단 알리자고 판단해서 장애물이라고 말하고 있었다.\\
Here's what was happening: the distance sensor would report something within 1 meter, the camera couldn't identify what it was, and the device decided it didn't know what it was but it was close so it should warn anyway — and called it an "obstacle."

그런데 사진을 보면 그 장애물의 정체가 뭔지 명확했다. 벽이었다. 그리고 바닥이었다.\\
But looking at the photos, it was obvious what that "obstacle" actually was. A wall. Or the floor.

골목길 사진 한 장은 오른쪽에 담벼락이 실제로 68cm 거리에 있었다. 기기는 정확하게 잰 것이다. 다만 그게 벽인 줄 몰랐을 뿐이다.\\
One alley photo had a wall genuinely 68cm away on the right. The device measured it correctly — it just had no idea it was a wall.

<!-- NEED: 담벼락이 68cm 거리로 잡힌 골목길 경고 사진 -->

이건 버그가 아니었다. 설계한 대로 정확히 동작한 것이라 100% 재현됐다. 벽 옆을 걸으면 무조건 울렸다.\\
This wasn't a bug. It was working exactly as designed, which is why it reproduced 100% of the time. Walk next to a wall, and it rings, no exceptions.

참고로 이 25분 동안 기기가 위험하다고 판정한 횟수는 6335번이었다. 초당 4번꼴이다. 중복을 걸러내는 장치가 250번으로 깎아준 게 그나마 저 숫자다.\\
For reference, over those 25 minutes the device flagged "danger" 6,335 times — about 4 times a second. The number that actually reached me, 250, was already after a deduplication filter cut it down.

---

## 2. 앞에 없는 물건 이름을 말하고 있었다

더 창피한 것도 발견했다. 경고할 때 말한 물건 이름과 그 순간 사진에 실제로 찍힌 것이 30번 중 24번, 80%가 달랐다.\\
I found something even more embarrassing. In 24 out of 30 warnings — 80% — the object name it announced didn't match what was actually in the photo at that moment.

| 말한 것 | 사진에 실제로 있던 것 |
|---|---|
| 사람 | 없음 |
| 버스 | 냉장고, 사과 |
| 그릇 | 없음 |
| 화병 | 없음 |
| "장애물" | 버스 ← 이건 반대로 틀림 |

원인은 단순했다. 기기는 1초에 15번 판단하는데 사진 인식은 그보다 느릴 때가 있다. 새 인식 결과가 아직 안 나왔으면 직전 결과를 그대로 쓰고 있었다.\\
The cause was simple. The device makes a judgment 15 times a second, but image recognition is sometimes slower than that. Whenever a new result wasn't ready yet, it just kept reusing the previous one.

그래서 이미 지나간 물건 이름을 계속 말하고 있었다. 사용자 입장에선 앞에 없는 물건 이름을 듣게 된다. 차라리 "앞에 뭔가 있어요"가 나았을 것이다.\\
So it kept announcing the name of something that was already gone. From the user's side, that means hearing the name of an object that isn't actually in front of them. Just saying "something's there" would have been better.

---

## 3. 바닥을 안 보이게 하려다 멈췄다

여기까지 오면 해결책은 뻔해 보였다. 바닥이 문제니까 바닥을 안 보게 하면 된다.\\
At this point the fix seemed obvious. The floor was the problem, so just stop looking at the floor.

거리 센서에는 그럴 수 있는 기능이 있다. 센서 안에는 빛을 받는 칸이 16×16으로 나뉘어 있는데, 그중 위쪽 절반만 쓰겠다고 지정할 수 있다. 부품을 덧대지 않고도 소프트웨어만으로 아래를 안 보게 만들 수 있는 것이다.\\
The distance sensor actually supports this. Its light-receiving area is split into a 16×16 grid, and you can tell it to use only the top half. That means blinding it to the ground purely in software, without adding any hardware.

준비까지 다 해뒀다. 어느 쪽이 아래인지 확인하는 프로그램도 만들어놨다.\\
I had it all ready to go. I'd even written a small program to check which side was "down."

그런데 실행하기 직전에 멈췄다.\\
And then, right before running it, I stopped.

---

## 4. 바닥은 노이즈가 아니라 신호였다

질문을 바꿔봤다. 바닥이 잡히는 게 정말 문제인가?\\
I changed the question. Was the floor actually being detected really a problem?

지팡이가 하는 일이 뭔가. 바닥을 두드리는 것이다. 앞에 턱이 있는지 계단이 있는지 뭐가 놓여 있는지 바닥을 짚어서 안다.\\
What does a cane actually do? It taps the ground. It tells you about a curb, a step, or something lying in the way by feeling the floor.

그런데 나는 지금 바닥이 잡혀서 시끄럽다며 그걸 지우려고 몇 주를 쓰고 있었다. 억제 장치를 만들고 임계값을 조정하고, 이번엔 센서 시야까지 자르려던 참이었다.\\
But I had spent weeks trying to erase exactly that — building suppression logic, tuning thresholds, and now about to cut the sensor's field of view too, all because floor detection was "too noisy."

바닥은 노이즈가 아니라 신호였다. 이걸 인정하니까 할 일이 오히려 줄었다.\\
The floor wasn't noise. It was signal. Once I admitted that, my to-do list actually got shorter.

| 없어진 일 | 이유 |
|---|---|
| 센서 시야 위쪽만 쓰기 | 새 방향과 정반대 |
| 바닥 경고 억제 로직 튜닝 | 억제할 이유가 없어짐 |
| "벽인가 장애물인가" 구분 | 애초에 구분할 필요가 없음 |

---

## 5. 지팡이를 대신하려다 접었다

지팡이를 대체하는 물건을 만들면 되는 거 아닌가?\\
Why not just build something that replaces the cane entirely?

며칠 고민하다 접었다. 이유는 기술이 아니라 고장 때문이다.\\
I thought about it for a few days and dropped it. Not because of the technology — because of failure modes.

오늘 데이터를 다시 보면, 카메라를 믿을 수 없었던 시간이 7.2%, 속도가 떨어져 비상 모드로 간 시간이 3.9%였다.\\
Looking at today's data again: the camera was untrustworthy 7.2% of the time, and the device dropped into emergency mode from slow frame rate 3.9% of the time.

보조 도구라면 괜찮은 숫자다. 지팡이가 받쳐주니까. 하지만 대체품이라면 100번 중 7번 눈을 감는 셈이다. 게다가 배터리가 있고, 부팅에 시간이 걸리고, 더우면 느려지고, 소리가 안 나는 날도 있었다.\\
As an assistive tool, those numbers are fine — the cane still backs it up. But as a replacement, that's closing your eyes 7 times out of 100. On top of that, it has a battery, takes time to boot, slows down in the heat, and some days the sound just doesn't come out.

지팡이는 고장이 없다.\\
A cane never breaks down.

그리고 더 중요한 게 있다. 도달 불가능한 목표를 잡으면 결과가 하나뿐이다. "안 됐다." 이미 그 근처까지 갔다 왔다.\\
And there's something more important. Chase an unreachable goal, and there's only one outcome: "it didn't work." I'd already been close enough to that outcome once.

그래서 목표 문장을 이렇게 정했다. 지팡이가 닿는 곳을 2~3미터 먼저 보고, 지팡이가 못 닿는 머리 높이까지 본다.\\
So I settled on a new mission statement: see 2 to 3 meters ahead of where the cane can reach, and see all the way up to head height, where the cane can never reach at all.

지팡이가 닿는 거리는 대략 1.2미터다. 그보다 앞을 미리 아는 건 지팡이가 못 하는 일이다. 대체가 아니라 먼저 알려주기다.\\
A cane's reach is roughly 1.2 meters. Knowing what's beyond that in advance is something a cane simply can't do. This isn't a replacement — it's an early warning.

---

## 6. 버튼 누르고 한 바퀴 도는 기능을 넣기로 했다

방향을 바꾸면서 새 기능 하나를 넣기로 했다. 버튼을 누르고 가만히 서서 한 바퀴 돌면, 기기가 주변에 뭐가 있는지 하나씩 말해주는 기능이다.\\
Along with this shift in direction, I decided to add one new feature: press a button, stand still, turn around once, and the device announces what's around you, one thing at a time.

```
[버튼 누름]
"주변을 확인합니다. 천천히 한 바퀴 도세요."
[회전]
"확인 완료. 네 개 찾았습니다."
"정면 2미터, 의자."
"오른쪽 45도, 사람."
"뒤쪽, 벤치 둘."
```

이 기능이 좋은 이유는 오늘 실패한 문제가 거의 다 사라진다는 것이다.\\
What makes this feature good is that almost every problem I hit today disappears with it.

| 오늘의 문제 | 이 기능에서는 |
|---|---|
| 1분에 10번 시끄러움 | 사용자가 물어봤을 때만 답함 |
| 없는 물건 이름을 말함 | 버튼 누른 그 순간에 새로 찍으면 됨 |
| 걸으면서 찍어 사진이 흐림 | 서 있으니 안 흐림 |
| 0.5초 안에 답해야 함 | 기다리는 중이니 1초 써도 됨 |

그리고 이건 "굳이 이걸 왜 써?"에 대한 답이 된다.\\
And this also answers the question "why would I even need this thing?"

걸어가면서 울리는 경고는 솔직히 지팡이와 하는 일이 겹친다. 하지만 여기가 어디고 주변에 뭐가 있는지는 지팡이가 못 하는 일이다. 낯선 공간에서 방향을 잡는 건 실제로 큰 어려움이라고 한다.\\
Honestly, warnings while walking overlap with what a cane already does. But knowing "where am I, and what's around me" is something a cane can't do at all. Orienting yourself in an unfamiliar space is apparently a real, significant difficulty.
---

## 7. 다음 진행할 내용

다음은 이 주변 확인 기능을 실제로 만드는 일이다. 일단 부품을 더 달지 않고, 회전하는 동안 걸린 시간만으로 방향을 대충 추정하는 방식부터 붙여볼 생각이다.\\
Next is actually building this surroundings-check feature. For now, I'm going to start without adding any new hardware — just estimating direction from how much time has passed during the turn.

그걸로 방향 구분이 충분히 되는지 밖에서 직접 돌아보며 확인하고, 부족하면 그때 방향 감지 부품을 추가하기로 했다. 그리고 지금 쓰고 있는 음소거 버튼도 짧게/길게 누르는 동작으로 나눠서 두 기능을 한 버튼에 넣어야 한다.\\
I'll go test outside whether that's precise enough to tell directions apart, and if it isn't, that's when the orientation sensor comes in. I also need to split the existing mute button into short-press and long-press actions so both features can live on the same button.
