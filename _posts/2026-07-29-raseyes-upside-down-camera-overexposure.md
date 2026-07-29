---
title: "RasEyes: 거꾸로 달린 카메라, 그리고 조용해졌지만 아무것도 못 보는 기기"
date: 2026-07-29 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [orangepi, camera, mipi, ov13855, object-detection, telemetry, monitoring, embedded, troubleshooting]
---

기기가 뭘 보고 무슨 판단을 했는지 들여다볼 수 있게 만들었더니, 그동안 몰랐던 문제 두 개가 곧바로 튀어나왔다.\\
I made it possible to look into what the device was seeing and deciding, and two problems I had never noticed immediately surfaced.

---

## 1. 기기가 뭘 봤는지 확인할 방법이 없었다

RasEyes는 머리 높이의 장애물을 감지해서 소리로 알려주는 기기다. 그런데 지금까지는 밖에 나가서 기기를 착용하고 걷다가 "방금 왜 울렸지?" 싶어도 확인할 길이 전혀 없었다. 경보가 제대로 울린 건지 헛울린 건지 판단할 근거가 없으니, 뭘 고쳐야 할지도 알 수가 없었다.\\
RasEyes is a device that detects head-height obstacles and alerts you with sound. Until now, though, when I was out walking with it on and wondered "why did it just beep?", there was no way to check. With no basis for judging whether an alert was correct or a false alarm, I couldn't tell what needed fixing either.

그래서 이번에는 기능을 늘리는 대신, 기기를 관찰할 수 있게 만드는 데 시간을 썼다. 위험 경보가 울리면 그 순간 전후의 영상을 사진 여러 장으로 남기도록 했다. 경보 전 5초와 경보 후 3초를 남기기 때문에, 나중에 열어보면 "아, 이 의자 때문에 울렸구나" 하는 판단이 가능해진다. 여기에 더해 속도(FPS)·온도·거리·경보 여부 같은 기기 상태를 1초에 한 번씩 기록하도록 했고, 이 기록을 명령어 한 줄로 PC에 가져와 또 한 줄로 통계를 뽑을 수 있게 정리했다.\\
So this time, instead of adding features, I spent the time making the device observable. When a hazard alert fires, it now saves the footage around that moment as a series of images — 5 seconds before the alert and 3 seconds after — so that opening them later lets me conclude "ah, it was that chair." On top of that, it records device state such as FPS, temperature, distance, and alert status once per second, and I set things up so I can pull those records to my PC with one command and generate statistics with another.

그리고 이 관찰 장치를 켜자마자, 예상하지 못했던 문제가 두 개 걸려 나왔다.\\
And the moment I turned this observation setup on, two problems I hadn't expected came out.

---

## 2. 정작 기기를 켜니 카메라가 없다고 했다

기기를 켜자마자 로그에 같은 에러가 끝도 없이 찍히고 있었다.\\
As soon as I powered the device on, the same error was being printed endlessly in the log.

```
카메라 프레임 캡처 실패 — 카메라 연결을 확인하세요
```

원인은 허무할 정도로 단순했다. 카메라를 다른 포트에 꽂아놓고 설정은 예전 그대로 두고 있었다. Orange Pi 5에는 카메라를 꽂을 수 있는 자리가 여러 개 있고 어느 자리에 꽂았는지를 설정 파일에 적어줘야 하는데, 설정에는 옛날 포트가 적혀 있으니 기기 입장에서는 비어 있는 자리를 계속 들여다보며 "카메라가 없다"고 외치고 있었던 셈이다.\\
The cause was almost absurdly simple. I had plugged the camera into a different port and left the configuration as it was. The Orange Pi 5 has several camera slots, and you have to record which one you used in a config file — but the config still pointed at the old port, so from the device's point of view it was staring at an empty slot and shouting "there's no camera."

단서는 엉뚱한 데서 나왔다. 카메라 모듈 안에는 초점을 맞추는 작은 모터가 같이 들어 있는데, 이 모터는 카메라 설정과 무관하게 항상 전원을 받는다. 그래서 카메라 본체는 조용한데 이 모터 하나만 엉뚱한 자리에서 혼자 신호를 보내고 있었고, 결국 모터가 보이는 자리가 곧 카메라가 꽂힌 자리였다. 설정을 그 자리로 바꾸고 재부팅하니 카메라가 그대로 살아났다.\\
The clue came from an unexpected place. The camera module contains a small focus motor, and that motor is powered regardless of the camera configuration. So while the camera body stayed silent, this one motor was signalling all by itself from a slot I wasn't looking at — and the slot where the motor showed up was exactly where the camera was plugged in. I switched the config to that slot, rebooted, and the camera came back to life.


<!-- NEED: 카메라를 실제로 꽂은 MIPI 포트 위치가 보이는 하드웨어 사진 -->

---

## 3. 사람이 화면을 가득 채웠는데 "아무것도 못 찾음"이었다

카메라가 살아났으니 경보를 일부러 한 번 울려봤다. 영상은 잘 저장됐다. 그런데 같이 저장된 판단 기록을 열어보니 이상했다. 사람이 화면을 가득 채우고 있는 장면인데 "아무것도 못 찾음"이라고 적혀 있었다.\\
Now that the camera was alive, I deliberately triggered an alert. The footage saved fine. But when I opened the decision log stored alongside it, something was off: in a frame where a person filled the entire screen, it read "nothing detected."

사진을 직접 열어보니 이유가 바로 보였다. 영상이 위아래로 뒤집혀 있었다. 사람을 알아보는 AI 모델은 똑바로 선 사람 사진으로 학습했기 때문에, 거꾸로 뒤집힌 사람은 잘 알아보지 못한다. 얼마나 차이가 나는지 궁금해서 저장된 사진을 그대로 놓고 한 번, 180도 돌려서 한 번 비교해봤다.\\
Opening the images made the reason obvious immediately: the video was flipped upside down. The person-detection model was trained on upright photos of people, so it struggles with people turned on their heads. Curious how big the difference was, I ran the saved images twice — once as-is, once rotated 180 degrees.

---

## 4. 분당 181번 울리던 경보는 29번으로 줄었다

어제 첫 야외 테스트에서 가장 큰 문제는 경보가 쉴 새 없이 울린 것이었다. 분당 181회, 거의 2초에 한 번씩 삐- 소리가 났으니 착용하고 걸어다닐 수준이 아니었다.\\
In yesterday's first outdoor test, the biggest problem was that the alerts never stopped. 181 times per minute — a beep roughly every 2 seconds — which is nowhere near wearable.

원인은 판단 방식에 있었다. 기기는 "1m 안에 뭔가 있으면 위험"이라고 보는데, 이건 켜졌다 꺼졌다 하는 상태값이다. 길을 걷다 보면 담벼락이든 지면이든 1m 안에 들어와 있는 게 정상이라, 위험 상태가 길게 유지되고 그동안 계속 울렸던 것이다. 그래서 위험한 상태인 동안 계속 울리는 대신 위험해진 순간에만 울리도록 바꿨더니, 분당 181회가 29회로 떨어졌다. 84% 줄어든 셈이고, 목표로 잡았던 "분당 30회 이하"도 넘겼다.\\
The cause was in how it judged. The device treats "something within 1m" as hazardous, but that's a state that toggles on and off. While walking, it's perfectly normal for a wall or the ground to be within 1m, so the hazard state stayed on for long stretches and it beeped the whole time. So I changed it to fire only at the moment the state becomes hazardous, rather than continuously while it is — and 181 per minute dropped to 29. That's an 84% reduction, and it clears the "under 30 per minute" target I had set.

---

## 5. 조용해진 것과 잘 보게 된 것은 다르다

여기서 만족하고 넘어갈 뻔했는데, 기기가 조용해졌다는 게 제대로 보고 있다는 뜻은 아니다. 아무것도 못 보고 있어도 조용하다. 그래서 오늘 저장된 사진들을 하나씩 직접 열어봤다.\\
I almost stopped there satisfied, but a quiet device doesn't mean a device that sees well. It's also quiet when it sees nothing at all. So I opened today's saved images one by one.

![과노출로 화면 전체가 하얗게 날아간 프레임](/images/20260729_overexposed.jpg)

*오늘 낮 산책로에서 저장된 사진. 왼쪽 나무 기둥 말고는 전부 하얗게 날아갔다.*\\
*A frame saved on a walking trail this afternoon. Everything except the tree trunk on the left is blown out to white.*

카메라 밝기 조절이 전혀 동작하고 있지 않았다. 실내 기준으로 한 번 맞춰놓고 그대로 고정된 상태라, 햇빛 아래로 나가면 이렇게 전부 하얗게 타버린다. 반대로 그늘이나 실내로 들어가면 이번엔 캄캄해서 아무것도 안 보인다. 사실상 카메라로는 아무것도 못 보고 있는 상태였다.\\
The camera's brightness control wasn't working at all. It had been set once for indoor conditions and stayed frozen there, so stepping into sunlight burns everything out to white like this. Step into shade or indoors and it goes pitch dark instead. Practically speaking, the camera was seeing nothing.

그러면 기기는 대체 무엇으로 판단하고 있었을까. RasEyes에는 카메라 말고 거리 센서가 하나 더 있다. 카메라가 못 볼 때를 대비한 예비 수단인데, 오늘은 이 예비 수단에만 의존한 시간이 96%였다. 문제는 거리 센서가 "뭐가 있는지"는 모르고 "얼마나 먼지"만 안다는 점이다. 앞이 사람인지 담벼락인지 바닥인지 구분이 안 되니, 산책로 바닥이든 석축이든 1m 안에 들어오면 전부 장애물로 치고 울린다. 오늘 울린 경보의 대부분이 이렇게 나갔다.\\
So what was the device judging with? RasEyes has a distance sensor in addition to the camera — a fallback for when the camera can't see. Today, 96% of the time it was running on that fallback alone. The problem is that the distance sensor knows "how far," not "what." It can't tell a person from a wall from the ground, so anything within 1m — trail surface, stone embankment — counts as an obstacle and triggers a beep. Most of today's alerts came out this way.

결국 경보를 줄인 건 증상을 가린 쪽에 가깝고, 진짜 원인인 카메라는 그대로 남아 있다.\\
In the end, cutting the alerts was closer to masking a symptom, and the real cause — the camera — is still there.

> ⚠️ 지금 상태로는 실제 보행 보조 용도로 쓰면 안 된다. 테스트 목적으로만 착용하고 있다.\\
> ⚠️ In its current state this must not be used as an actual walking aid. I'm wearing it for testing purposes only.

---

## 6. 다음 진행할 내용

다음에는 카메라 밝기가 주변 환경에 따라 자동으로 맞춰지도록 손보는 것이 1순위다. 그게 되고 나서야 거리 센서에만 의존하지 않는 상태가 되고, 그때 거리 센서 쪽 판단 기준을 다듬을 생각이다.\\
Next up, the top priority is making the camera's brightness adjust automatically to its surroundings. Only once that works will the device stop leaning entirely on the distance sensor, and that's when I plan to refine the distance sensor's decision thresholds.
