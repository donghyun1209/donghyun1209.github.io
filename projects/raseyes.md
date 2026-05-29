---
layout: page
title: "RasEyes (라즈아이즈)"
permalink: /projects/raseyes/
---

> **웨어러블 엣지 AI 보행 보조 시스템 (Wearable Top Obstacle Detection System for the Blind)** — 시각장애인의 상단 사각지대 장애물을 실시간으로 감지하여 청각적 경고를 제공하는 PoC 디바이스. (A PoC device that provides audible warnings by detecting blind people's top blind spot obstacles in real time.)

---

## 문제 정의

지팡이는 바닥 장애물만 탐지할 수 있어, **가슴·머리 높이에 위치한 장애물**(간판, 나뭇가지, 트럭 적재함 등)에 의한 충돌 사고 위험이 상존합니다. 
The cane can only detect floor obstacles, and there is always a risk of collision due to obstacles located at chest and head height (signboards, branches, truck loaders, etc.).

## 솔루션

카메라 비전 AI + ToF 거리 센서를 결합한 체스트 스트랩형 웨어러블 기기로, **두 손을 자유롭게** 유지하면서 전방 상단 장애물을 감지해 이어폰으로 경고음을 전달합니다.
A chest strap-type wearable device that combines camera vision AI + ToF distance sensor to detect front top obstacles while **keeping both hands free** and delivering a warning sound to the earphone.

## 시스템 구성

```
[카메라] ──→ YOLOv8 Nano (RKNN/NPU)
                    ↓ 바운딩 박스
[ToF 센서] ──→ 거리 측정          → 센서 퓨전 → 오디오 피드백
                    ↓ (I2C)             (둘 다 1.5m 이내)
              Orange Pi 5 (RK3588S)
```

| 구성 요소 | 사양 |
|-----------|------|
| **메인 보드** | Orange Pi 5 (4GB, RK3588S + NPU 6TOPS) |
| **AI 모델** | YOLOv8 Nano → RKNN 변환 |
| **거리 센서** | VL53L1X ToF (I2C) |
| **카메라** | MIPI CSI |
| **출력** | 이어폰 |
| **OS** | Armbian/Ubuntu + systemd 데몬 |

## 목표 지표 (KPIs)

| 지표 | 목표 |
|------|------|
| 시스템 지연 | < 500ms |
| 추론 속도 | ≥ 15 FPS |
| 감지 재현율 | > 95% (2m 이내) |
| 오탐지 | < 1회/분 |
| 열 스로틀링 | 사용 시간의 5% 미만 |

## 개발 로그

아래 포스트들에서 개발 과정을 기록하고 있습니다.

{% assign raseyes_posts = site.posts | where_exp: "post", "post.categories contains 'RasEyes'" %}
{% if raseyes_posts.size > 0 %}
{% for post in raseyes_posts %}
- [{{ post.title }}]({{ post.url }}) <small>{{ post.date | date: "%Y-%m-%d" }}</small>
{% endfor %}
{% else %}
*아직 작성된 포스트가 없습니다. 곧 업데이트될 예정입니다.*
{% endif %}
