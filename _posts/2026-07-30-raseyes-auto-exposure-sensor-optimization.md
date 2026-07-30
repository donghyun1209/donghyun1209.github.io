---
title: "RasEyes: 카메라 자동 밝기 조절 및 거리 센서 최적화"
date: 2026-07-30 09:00:00 +0900
categories: [RasEyes, Embedded System]
tags: [camera, exposure, sensor, optimization, embedded, raseyes]
---

오늘은 카메라가 스스로 눈을 뜨게 만들고 거리 센서의 고질적인 문제를 해결하는 작업을 했다.\\
Today, I worked on making the camera open its eyes automatically and solving the chronic problem of the distance sensor.

## 1. 카메라에 자동 밝기 조절 기능이 없었다

카메라는 보통 어두운 곳과 밝은 곳을 오갈 때 스스로 적응한다. 그래서 처음에는 당연히 그 기능을 켜기만 하면 될 줄 알았다.\\
Cameras usually adapt themselves when moving between dark and bright places. So at first, I thought I just needed to turn on that feature.

하지만 기기에 붙은 카메라에는 사람이 직접 노출과 감도를 정해주는 스위치만 있을 뿐, 알아서 맞춰주는 기능이 아예 없었다. 심지어 감도가 최대치로 고정되어 있어서, 실내용으로 눈을 크게 뜬 채 햇빛 아래로 나가니 화면이 하얗게 타버릴 수밖에 없었다.\\
However, the camera attached to the device only had switches for manual exposure and sensitivity, and no automatic adjustment feature at all. Even worse, the sensitivity was fixed at the maximum, so it was like going out in the sunlight with eyes wide open for indoors, inevitably burning the screen white.

<!-- NEED: 햇빛에 하얗게 타버린 카메라 캡처 이미지 -->

## 2. 결국 직접 자동 밝기 조절을 만들었다

없으면 직접 만들어야 했다. 처음엔 화면의 평균 밝기만 보면 될 줄 알았는데, 화면 절반이 햇빛에 새하얗게 타고 나머지 절반이 그늘이라 새까맣게 되면 평균값은 '정상'으로 나오는 함정이 있었다.\\
If it's not there, I had to make it myself. At first, I thought I only needed to check the average brightness of the screen, but there was a trap where the average value appeared 'normal' if half the screen was burnt white by sunlight and the other half was pitch black in the shade.

그래서 하얗게 타버린 점이 몇 퍼센트인지 함께 계산해서 기준을 넘으면 무조건 어둡게 만들도록 고쳤다. 그리고 밝기를 조절하라고 명령한 뒤에는 바로 반영되지 않는다는 점을 고려해, 화면이 계속 깜빡이지 않도록 잠시 기다리는 시간도 넣었다.\\
So I fixed it to calculate the percentage of burnt white spots as well, unconditionally darkening it if it exceeds the standard. And considering that the command to adjust brightness is not reflected immediately, I also added a waiting time so the screen wouldn't keep blinking.

## 3. 카메라가 멀쩡한데 볼 게 없는 상황을 구분했다

기존에는 카메라가 안 보이는 상황과 멀쩡히 보고 있는데 앞이 빈 상황을 똑같이 취급했다. 그러다 보니 앞이 멀쩡히 보이는데도 거리 센서에 바닥이나 벽이 1.5m 안에 들어오면 무조건 알림이 울렸다.\\
Previously, the situation where the camera couldn't see and the situation where it could see but the front was empty were treated the same. Because of this, even when the front was clearly visible, the alarm would unconditionally go off if the floor or a wall came within 1.5m of the distance sensor.

걸어 다니면 1분에 10번씩 1.5m 경계선을 왔다 갔다 하는데, 그때마다 계속 울리니 정신이 없었다. 그래서 카메라가 멀쩡히 보고 있는데 아무것도 못 찾은 거라면 1.5m가 아닌 1m 안에 들어왔을 때만 울리도록 바꿨다.\\
Walking around meant crossing the 1.5m boundary 10 times a minute, and it was so distracting because it kept ringing every time. So I changed it so that if the camera was clearly looking but couldn't find anything, it would only ring when it came within 1m instead of 1.5m.

다만, 카메라가 나뭇가지나 간판처럼 배우지 못한 물체를 진짜로 못 알아본 것일 수도 있기 때문에 1m 안전장치는 꼭 남겨뒀다.\\
However, I strictly kept the 1m safety device because the camera might have genuinely failed to recognize unlearned objects like branches or signs.

여기서 한 가지 아찔했던 건, 카메라가 버벅일 때 인식 결과를 강제로 비워버리는 기존의 안전장치와 이 규칙이 꼬일 뻔했다는 것이다. 고장 나서 결과가 비워진 걸 '앞에 아무것도 없네'로 착각해서 가장 위험한 순간에 경고를 꺼버릴 뻔했다. 카메라가 멈췄을 때도 제대로 위험 상황으로 인지하도록 조건을 꼼꼼하게 다시 잡았다.\\
One dizzying moment here was that this rule almost got tangled with the existing safety device that forcibly clears the recognition result when the camera stutters. I almost mistook the cleared result due to a breakdown for 'there is nothing in front', turning off the warning at the most dangerous moment. I meticulously reset the conditions to properly recognize it as a dangerous situation even when the camera stops.

## 4. 거리 센서의 가짜 평균 내기를 고쳤다

거리 센서의 값이 튀는 걸 막으려고 최근 3번의 평균을 쓰도록 해뒀는데, 알고 보니 센서가 1초에 5번 재는 동안 기기가 1초에 15번 물어보고 있었다. 결국 똑같은 값을 세 번 받아서 평균을 내는 꼴이라 아무 의미가 없었다.\\
I set it to use the average of the last 3 times to prevent the distance sensor values from bouncing, but it turned out the device was asking 15 times a second while the sensor was measuring 5 times a second. Eventually, it was meaningless because it was just taking the exact same value three times and averaging it.

센서 값이 갱신될 때만 카운트를 세도록 고쳤다. 값이 안정적으로 바뀌었지만 반응이 0.6초 정도로 느려졌기 때문에 밖에서 테스트하며 밸런스를 맞춰봐야 한다.\\
I fixed it to count only when the sensor value is updated. The value became stable, but the response slowed down to about 0.6 seconds, so I need to balance it out while testing outside.

## 5. 이벤트 클립 저장 최적화

HIGH 경보가 발생했을 때 앞뒤로 여러 장의 사진을 저장하던 것을, 트리거 순간의 프레임 1장과 메타데이터만 저장하도록 뜯어고쳤다. 용량도 아끼고 분석하기도 훨씬 편해졌다.\\
I overhauled the system to save only 1 frame at the moment of the trigger and its metadata, instead of saving multiple photos back and forth when a HIGH alarm occurred. It saves capacity and makes analysis much easier.

<!-- NEED: 최적화된 trigger.jpg 샘플 이미지 -->