---
title: "RasEyes: 길 안내를 붙였다가 위험한 버그 두 개를 발견했다"
date: 2026-08-30 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [raseyes, navigation, bluetooth, tts, obstacle-detection, unit-test, orangepi5, embedded]
---

지금까지 이 기기는 "앞에 장애물이 있다"는 것만 알려줬다. 어디로 가야 하는지는 알려주지 못했다. 오늘은 스마트폰에서 목적지를 검색하면 기기가 "50미터 앞에서 우회전하세요"라고 말해주는 길 안내 기능을 붙였다.\\
Until now, this device could only say "there's an obstacle ahead" — it never told you where to go. Today I hooked up walking navigation, so searching a destination on the phone makes the device say things like "turn right in 50 meters."

폰에서 목적지를 검색해 도보 경로를 받아오고, 그걸 블루투스로 기기에 넘기고, 기기가 그 지시를 문장으로 풀어 음성 안내하고, 걷는 동안 자동으로 다음 지시로 넘어가는 것까지 오늘 하루에 다 붙였다.\\
Searching a destination and fetching the walking route on the phone, sending it to the device over Bluetooth, having the device turn that into a spoken sentence, and auto-advancing to the next instruction while walking — all of it went in today.

---

## 오렌지파이에 블루투스가 없었다

기기(오렌지파이 5)에 당연히 블루투스가 있을 줄 알았는데 없었다.\\
I assumed the device (an Orange Pi 5) had Bluetooth built in. It didn't.

집에 굴러다니던 블루투스 동글을 꽂는 걸로 일단 해결했다.\\
I fixed it, for now, by plugging in a spare Bluetooth dongle I had lying around.

---

## 폰은 문장 대신 네 글자짜리 암호를 보낸다

"50미터 앞에서 우회전하세요" 같은 문장을 통째로 보내면 편하겠지만, 실제로 폰이 보내는 건 `R|50`처럼 딱 네 글자다. 이유가 두 가지 있다.\\
It would be simpler to just send the whole sentence "turn right in 50 meters," but what the phone actually sends is four characters like `R|50`. There are two reasons for that.

블루투스로 한 번에 보낼 수 있는 글자 수가 20자로 제한돼 있어서 문장을 통째로 보내면 넘친다. 게다가 기기 음성이 영어로 고정돼 있다. 한국어 목소리를 붙여봤는데 어색해서 영어로 결정했더니, 길 안내 API가 주는 한국어 문장을 그대로 읽힐 수가 없게 됐다.\\
Bluetooth caps each transfer at 20 characters, so a full sentence overflows. On top of that, the device's voice is fixed to English — I tried Korean TTS and it sounded awkward, so I stuck with English, which meant I couldn't just read out the Korean sentences the navigation API returns.

그래서 폰은 뜻만 담은 짧은 코드를 보내고, 기기가 그걸 받아서 "Turn right in 50 meters" 같은 문장으로 조립해서 말하게 만들었다.\\
So the phone sends a short code carrying just the meaning, and the device assembles it into a sentence like "Turn right in 50 meters" before speaking.

---

## 모르는 지시를 "직진하세요"라고 말해버리고 있었다

다 만들고 나서 제대로 동작하는지 점검하다가 아찔한 버그를 하나 찾았다.\\
While checking that everything actually worked after building it, I found a bug that made my stomach drop.

길 안내 API가 주는 지시는 30가지쯤 되는데, 폰이 처리할 수 없는 지시가 오면 화면에 주황색 `?` 표시를 남기도록 해뒀다. "나는 이걸 모른다"는 뜻이다.\\
The navigation API sends roughly thirty different instruction types, and I'd set the phone to leave an orange `?` mark on screen whenever an instruction it couldn't handle came in — meaning "I don't understand this one."

그런데 기기는 이 `?`를 받으면 "Proceed(진행하세요)"라고 말해버리고 있었다. 진행하라는 말은 사실상 직진하라는 뜻이다. 만약 우회전해야 하는 지점에서 "진행하세요"라고 안내한다면, 시각장애인 사용자에게는 그대로 사고로 이어질 수 있는 문제다.\\
But when the device received that `?`, it said "Proceed" — which is effectively an instruction to go straight. If that happened at a spot where the user actually needed to turn right, telling a visually impaired user to "proceed" could lead directly to an accident.

폰 쪽은 원칙(모르면 모른다고 표시한다)을 지켰는데, 기기 쪽의 기본값 처리 한 줄 때문에 그 원칙이 무너져 있었다.\\
The phone had honored the principle — flag it when you don't understand — but a single default-value line on the device side had quietly broken that principle.

그래서 알 수 없는 지시가 오면 "Caution, unknown instruction in 50 meters(주의, 50미터 앞 알 수 없는 지시)"라고 말하도록 고쳤다. 거리는 알려주되 판단은 사람에게 넘기는 쪽으로 바꿨다.\\
I changed it so that an unknown instruction now says "Caution, unknown instruction in 50 meters" — the distance is still announced, but the judgment call is left to the person.

<!-- NEED: 앱 화면에 표시되는 주황색 "?" 지시 아이콘 스크린샷 -->

---

## 장애물 경보가 길 안내 멘트에 밀려 조용해지고 있었다

기기의 경고는 크게 두 단계다. 1미터 이내는 "가까움(HIGH)", 1.5미터 이내는 "주의(MID)"다. 그리고 절대 규칙이 하나 있다. 장애물 경보는 언제나 길 안내를 끊고 먼저 말해야 한다는 것이다. 길을 찾는 것보다 부딪히지 않는 게 훨씬 중요하니까.\\
The device has two obstacle warning levels: "HIGH" inside 1 meter and "MID" inside 1.5 meters. And there's one absolute rule — an obstacle warning must always interrupt navigation and speak first, because not walking into something matters far more than finding the way.

그런데 실제로는 HIGH만 이 규칙을 지키고 있었다. MID 경보는 길 안내 멘트가 나오는 중이면 조용히 묻혀버렸다. 이 기기는 같은 경고를 반복해서 삑삑거리지 않도록 한 번 나온 경고는 다시 내지 않게 설계돼 있는데, 그 설계 때문에 한 번 묻힌 MID 경고는 영영 다시 나오지 않았다. 길 안내를 듣는 동안 옆에 있는 기둥을 계속 인지하지 못하는 셈이다.\\
In practice, only HIGH followed that rule. A MID warning would get silently swallowed whenever a navigation sentence was already playing. And because the device is designed not to repeat a warning it already gave — so it doesn't nag about the same thing over and over — a MID warning that got swallowed once simply never came back. Which meant the user could stay completely unaware of a pillar right next to them for as long as navigation kept talking.

고치려고 기기가 "지금 자기가 무슨 멘트를 하고 있는지"를 기억하게 만들었다. 지금 말하는 게 길 안내라면 장애물 경보가 끊고 들어간다. 지금 말하는 게 다른 장애물 경보라면 예전처럼 기다린다. 끊긴 길 안내는 경보가 끝난 뒤 다시 이어서 말해준다.\\
The fix was to make the device remember what kind of thing it's currently saying. If it's mid-navigation, an obstacle warning cuts in. If it's already mid-warning, it waits, same as before. And an interrupted navigation sentence resumes once the warning finishes.

---

## 테스트 코드가 하나도 없어서 이걸 못 보고 있었다

이런 위험한 버그를 개발 중에 못 잡았던 근본적인 이유는 단순했다. 자동으로 확인해주는 테스트가 단 하나도 없었다. 다 만들고 눈으로 훑어본 뒤 "잘 되네" 하고 넘어갔던 것이다.\\
The reason these dangerous bugs slipped through during development was simple — there wasn't a single automated test for this part. I'd built it, eyeballed it, said "looks fine," and moved on.

그래서 판단하는 핵심 로직만 따로 분리했다. 절전 모드 판단이나 카메라 정지 판단처럼 이미 분리해뒀던 코드와 같은 형태로 맞췄다. 분리하고 나니 테스트를 붙일 수 있게 됐고, 새로 28개를 추가했다. 어제까지 337개였던 테스트가 오늘 365개가 됐다.\\
So I pulled the decision logic out on its own, matching the shape of code I'd already separated out before — like the power-save and camera-freeze judgment logic. Once it was separated, I could finally write tests against it, and added 28 new ones. The suite went from 337 tests yesterday to 365 today.

이 리팩토링 과정에서 버그를 하나 더 잡았다. 끊긴 길 안내를 나중에 다시 말하려고 임시로 저장해두는데, 그사이에 새 지시가 도착하면 당연히 최신 걸로 갱신해야 한다. 낡은 지시가 최신 지시를 덮어쓰도록 짜여 있던 걸 미리 발견하고 고쳤다.\\
This refactor caught one more bug along the way. An interrupted navigation sentence gets stashed to resume later — and if a new instruction arrives in the meantime, the stash obviously needs to update to the latest one. It had been wired the other way around, letting a stale instruction overwrite a fresh one, and I caught and fixed it before it became a real problem.

---

## 걸어가면 알아서 다음 지시로 넘어간다

원래 앱에는 "첫 단계 전송" 버튼 하나뿐이었다. 아무리 걸어도 다음 길 안내로 넘어가지 않아서, 실제로 밖에서 걸으면서 테스트하는 게 불가능했다.\\
Originally the app had exactly one button: "send first step." No matter how far you walked, it never advanced to the next instruction — which made it impossible to actually field-test by walking around outside.

이제는 위치를 계속 추적하면서, 지시 지점 60m 이내에 들어오면 한 번 미리 말해주고, 15m 이내에 들어오면 그 지점을 통과했다고 보고 다음 지시로 넘어간다.\\
Now it tracks location continuously — announcing an upcoming instruction once you're within 60 meters of it, and treating it as passed and advancing to the next one once you're within 15 meters.

여기서 하나 신경 쓴 부분이 있다. 경로 데이터에 적힌 거리는 "지금 남은 거리"가 아니라 "그 구간 전체 길이"다. 이걸 그대로 보내면 60미터 앞에서 엉뚱한 숫자를 말하게 된다. 그래서 기기에 멘트를 보내기 직전에 현재 위치 기준으로 거리를 다시 계산해서 정확한 숫자를 전달하도록 고쳤다.\\
One thing I had to watch for: the distance value in the route data isn't "distance remaining" — it's the total length of that leg of the route. Sending that as-is would make the device announce the wrong number at 60 meters out. So I made it recompute the distance from the current position right before sending the sentence to the device, so the number it says is actually correct.
