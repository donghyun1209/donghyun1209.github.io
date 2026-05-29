---
title: "오렌지파이 5(Orange Pi 5) 우분투 OS 플래싱 삽질"
date: 2026-05-29 18:00:00 +0900
categories: [Embedded System, Orange Pi]
tags: [orangepi, ubuntu, macos, troubleshooting, embedded]
---

오렌지파이 5(4GB RAM) 보드를 구동하기 위해 맥북 에어 환경에서 128GB 고성능 마이크로 SD카드에 **Ubuntu 22.04 Jammy Server** OS를 빌드하면서 겪은 험난한 트러블슈팅 기록이다.
This is a bumpy troubleshooting record I went through building the **Ubuntu 22.04 Jimmy Server** OS on a 128GB high-performance micro SD card in a MacBook Air environment to run an Orange Pie 5 (4GB RAM) board.

결론부터 말하자면, 보드나 SD카드가 아니라 **'저가형 SD카드 리더기'**가 원인이었다.
In conclusion, the cause was not the SD card, but the **'low cost SD card reader'**.

---

## 1. 1차 시도 (1st try)

가장 대중적인 플래싱 툴인 `balenaEtcher`를 이용해 압축을 푼 `.img` 파일을 구우려고 시도했다. 그러나 게이지가 채 시작되기도 전에 에러가 발생했다.
The attempt was made to bake an uncompressed '.img' file using the most popular flushing tool, 'balena Etcher.' However, the error occurred before the gauge even started.

```
writer process ended unexpectedly
```

![balenaEtcher 에러](/images/balenaEtcherError.png)

macOS의 깐깐한 보안 정책 때문에 Etcher가 디스크 깊숙한 곳에 접근하는 것이 차단된 것으로 판단하여, 시스템 설정에서 **[전체 디스크 접근 권한]**을 부여하고 에디터를 재실행했으나 동일한 증상이 반복되었다.
Determined that macOS's strict security policy prevented Etcher from accessing deep disks, the system setup gave **[Full Disk Access]** and ran the editor again, but the same symptoms were repeated.

---

## 2. 2차 시도 (2nd try)

Etcher의 버그를 의심하여 더 안정적인 `Raspberry Pi Imager`로 툴을 전환했다. 이번에는 게이지가 올라가는 듯했으나 중간에 멈춰버렸다.
Suspecting Etcher's bug, I switched the tool to a more stable 'Raspberry Pi Imager'. This time the gauge seemed to be going up, but it stopped in the middle.

```
Write stalled
```

![RPi Imager 프리징](/images/RPiImagerError.png)

디스크 파티션이 꼬여 강제 포맷을 시도했으나, 이마저도 `Waiting for partitions to activate` 단계에서 무한 프리징에 걸렸다. 결국 디스크 유틸리티 앱을 강제 종료하고 맥북을 재부팅해야 했다.
The disk partition was twisted and attempted to format, but even this was infinitely freaked out in the "Waiting for parts to activate" stage. Eventually, the disk utility app had to be forcibly shut down and the MacBook rebooted.

---

## 3. 범인 검거: 다이소 리더기의 한계 (Limitations of Daiso Reader)

두 가지 플래싱 툴과 OS 포맷 프로세스가 모두 특정 구간에서 뻗어버리는 현상을 분석한 결과, **물리적 연결 칩셋의 과열**이 원인임을 알아냈다.
After analyzing both flushing tools and OS formatting processes stretching over a specific section, I found that **overheating of the physical connection chipset** was the cause.

현재 사용 중인 임시방편용 **다이소 가성비 리더기**가 문제였다.
The issue was the Daiso Cost-Performance Reader for temporary measures currently in use.

---

## 4. 해결 (Solution)

이마트로 달려가 리더기를 새로 공수해왔다.
I ran to E-Mart and got a new device.

---

## 5. 요약 및 교훈

임베디드 보드 OS 플래싱처럼 대용량 연속 쓰기 작업을 할 때 **싸구려 리더기는 전력 부족과 과열로 무조건 뻗는다.**
When performing large continuous write operations like embedded board OS flashing **cheap readers have some problems..**.
