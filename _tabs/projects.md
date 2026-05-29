---
layout: page
icon: fas fa-folder-open
order: 5
title: Projects
---

진행 중이거나 완료된 개인 프로젝트들을 모아둔 페이지입니다.
It is a page that collects personal projects that are in progress or completed.

---

## 진행 중

### [RasEyes (라즈아이즈)](/projects/raseyes/)

> 시각장애인을 위한 웨어러블 상단 장애물 감지 시스템 (Wearable Top Obstacle Detection System for the Blind)

지팡이가 탐지하지 못하는 가슴·머리 높이 장애물을 카메라 AI와 ToF 센서로 감지하여 이어폰으로 경고음을 전달하는 엣지 AI 보행 보조 기기 PoC.
An edge AI walking assist device PoC that detects chest and head height obstacles that cane cannot detect with camera AI and ToF sensors and delivers warning sounds through earphones.

| 항목 | 내용 |
|------|------|
| **하드웨어** | Orange Pi 5 (RK3588S + NPU 6TOPS) |
| **AI 모델** | YOLOv8 Nano → RKNN |
| **센서** | VL53L1X ToF + MIPI 카메라 |
| **목표 지연** | < 500ms, 15 FPS 이상 |

[→ 프로젝트 상세 보기](/projects/raseyes/)
