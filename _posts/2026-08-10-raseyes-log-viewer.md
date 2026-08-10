---
title: "RasEyes: 흩어진 로그와 사진을 한 화면에 모았다"
date: 2026-08-10 09:00:00 +0900
categories: [RasEyes, Web]
tags: [raseyes, log-viewer, dashboard, calendar, timeseries, kpi, visualization, ux]
---

기기는 밖에서 걸을 때마다 1초에 한 줄씩 기록을 남긴다. 속도, 온도, 거리, 밝기 같은 것들이다. 경고가 울린 순간에는 사진도 한 장 따로 저장한다.\\
Every time the device is out walking, it writes one log line per second — speed, temperature, distance, brightness. And whenever a warning fires, it also saves a photo separately.

그런데 이걸로는 알 수 없는 게 있었다. 언제 그랬는지, 10분 내내 그랬는지 아니면 30초만 그랬는지. 그때 뭐가 보였는지, 사진은 다른 폴더에 있으니까. 여러 숫자가 같이 움직였는지, 속도가 떨어질 때 반응도 같이 느려졌는지.\\
But they couldn't tell me things I actually needed to know: when it happened — for ten minutes straight, or just thirty seconds? What was actually visible at that moment — the photo lived in a different folder. Did several numbers move together — did reaction time also slow down whenever speed dropped?

사진은 사진대로 흩어져 있었다. 그래서 오늘은 로그를 보는 화면을 만들었다.\\
And the photos were just scattered on their own. So today I built a screen for looking at all of this at once.

![로그 뷰어 전체 화면](images/viewer_overview.png)

왼쪽이 달력, 오른쪽이 그날 기록이다. 화면을 네 부분으로 나눴다.\\
Calendar on the left, that day's record on the right. I split the screen into four parts.

---

## 1. 빨간 점만 따라가면 문제 있던 날로 갈 수 있었다

![달력](images/viewer_calendar.png)

기록이 있는 날만 파랗게 칠해지고, 아래에 점이 하나씩 붙는다.\\
Only days with a recorded session get colored in blue, and each one gets a small dot underneath.

| 점 | 뜻 |
|---|---|
| 🔵 파랑 | 이날 경고 사진이 찍혔다 |
| 🟡 노랑 | 눈여겨볼 지표가 있다 |
| 🔴 빨강 | 목표치를 못 지킨 게 있다 |

날짜를 누르면 그날의 기록이 아래에 뜬다. 굳이 하루하루 로그 파일을 열어볼 필요가 없어졌다. 빨간 점만 따라가면 문제 있던 날로 바로 갈 수 있다.\\
Clicking a date pulls up that day's record below it. I no longer have to open log files one by one — just following the red dots takes me straight to the days something went wrong.

---

## 2. 카드 하나에 위반 이유까지 적기로 했다

![세션 KPI](images/viewer_kpi.png)

한 번 켜서 끌 때까지를 한 묶음으로 보고, 지표 14개를 카드로 보여준다. 숫자만 던져주면 결국 다시 계산해야 한다는 걸 알아서, 목표를 못 지키면 카드 테두리에 색이 들어오고 이유가 글로 붙게 만들었다.\\
Each power-on-to-power-off session gets grouped together and shown as 14 metric cards. I knew that just dumping raw numbers would mean re-deriving the meaning every time, so whenever a target is missed, the card border lights up and the reason gets written out in words.

> ⚠ KPI 위반 / ⚠ 모션블러 의심 / ⚠ 비전 실명 구간 많음

숫자를 보고 "이게 문제인가?"를 판단하던 걸, 화면이 대신 판단해서 알려주는 쪽으로 바꾼 셈이다.\\
Instead of staring at a number and asking myself "is this actually a problem?", the screen now makes that call and tells me directly.

---

## 3. 세로선 하나로 모든 그래프가 같이 움직였다

![시계열 그래프](images/viewer_charts.png)

마우스를 올리면 세로선 하나가 모든 그래프를 관통하고, 그 시각의 값이 전부 상자에 뜨도록 만들었다. 이제야 "속도가 떨어진 그 순간에 반응도 느려졌나"를 눈으로 바로 확인할 수 있게 됐다.\\
I made it so hovering draws a single vertical line through every chart at once, with all the values at that moment popping up in a box. Now I can actually see, at a glance, whether reaction time slowed down at the exact moment speed dropped.

맨 아래 줄은 경고 기록이다. 연한 파란 띠는 위험하다고 판단한 구간, 빨간 세로줄은 실제로 소리를 낸 순간, ▲는 사진이 찍힌 순간이다. ▲를 누르면 그 사진으로 바로 넘어간다.\\
The bottom row is the warning timeline. A light blue band marks a stretch judged dangerous, a red vertical line marks the moment it actually made a sound, and a ▲ marks the moment a photo was taken — clicking it jumps straight to that photo.

---

## 4. 사진은 결정적인 한 장만 남기기로 했다

![클립 화면](images/viewer_clip.png)

경고가 울린 순간의 사진이다. 기기가 찾아낸 물건에는 파란 상자가 씌워져 있다.\\
This is the photo from the moment a warning fired, with the object the device detected boxed in blue.

오른쪽에는 그때의 상황이 적힌다. 위 사진은 편의점 냉장고를 93.8cm 앞에서 만난 것이고, 병 9개와 냉장고를 찾아냈다.\\
The situation at that moment gets written out on the right. The photo above is of a convenience store fridge encountered 93.8cm away, with 9 bottles and the fridge itself detected.


