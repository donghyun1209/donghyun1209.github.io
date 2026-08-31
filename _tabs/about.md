---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

# 황동현 (DongHyun Hwang)

> 임베디드 리눅스와 온디바이스 AI 추론에 관심 있는 대학생. RasEyes 프로젝트로 카메라·센서 기반 엣지 AI 파이프라인을 기획부터 온디바이스 배포까지 직접 구현했습니다.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이름 | 황동현 (DongHyun Hwang) |
| 이메일 | 1209ghkdehdgus@gmail.com |
| GitHub | [donghyun1209](https://github.com/donghyun1209) |
| 블로그 | [donghyun1209.github.io](https://donghyun1209.github.io) |
| LinkedIn | [황동현](https://www.linkedin.com/in/%EB%8F%99%ED%98%84-%ED%99%A9-56905a3b0/) |

## 학력

- **단국대학교** 모바일시스템공학과 3학년
- 2026년 2학기 프랑스 교환학생 예정 (1개 학기)

## 목표

임베디드 시스템 + 엣지 AI 융합 엔지니어

## 관심 분야

- 임베디드 시스템 (MCU, RTOS, 펌웨어)
- 엣지 AI / 모델 경량화 / 추론 최적화
- IoT

## 기술 스택

| 분야 | 기술 |
|------|------|
| AI/DL | PyTorch, CNN, YOLOv8n, RKNN (INT8 양자화) |
| 임베디드 | Orange Pi 5, HAL, systemd, Linux (Ubuntu) |
| 인프라 | Docker |
| 언어 | Python |

## 수상

- **DKU 프로그래밍(DSPC) 경진대회** 장려상

## 자격증 / 수료증

| 과정 | 발행처 | 발행일 |
|------|--------|--------|
| Deep Learning Specialization | DeepLearning.AI | 2026.04 |
| PyTorch for Deep Learning | DeepLearning.AI | 2026.04 |
| Convolutional Neural Networks | DeepLearning.AI | 2026.03 |
| Structuring Machine Learning Projects | DeepLearning.AI | 2026.03 |
| PyTorch: Techniques and Ecosystem Tools | DeepLearning.AI | 2026.03 |
| Improving Deep Neural Networks | DeepLearning.AI | 2026.03 |
| PyTorch: Fundamentals | DeepLearning.AI | 2026.02 |
| Neural Networks and Deep Learning | DeepLearning.AI | 2026.02 |
| Docker 입문/실전 | 인프런 | 2026.02 |

## 프로젝트

### [RasEyes (라즈아이즈)](/projects/raseyes/) — 완료 (2026.05 ~ 2026.08)

**시각장애인을 위한 웨어러블 상단 장애물 감지 시스템 (PoC)**

- 지팡이가 탐지하지 못하는 가슴·머리 높이 장애물을 카메라 AI + ToF 센서로 감지 → 이어폰 경고음으로 안내
- 체스트 스트랩형 폼팩터, 양손 자유
- 추가 기능: 버튼 한 번으로 주변을 탐색하는 360도 스캔 모드, 목적지 길안내
- 365개 테스트 케이스로 검증

**기술 스택**: Python · Ubuntu 22.04 · YOLOv8n (INT8 양자화) · RKNN 기반 NPU 추론 (Orange Pi 5 HAL 포팅) · VL53L1X ToF 센서 · Docker · systemd

**성과**
- 개발 기간 약 3개월(2026.05.29 ~ 2026.08.31), 블로그 기술 일지 12편 작성
- 텔레메트리·로그 뷰어 구축으로 숨은 버그 다수 발견 및 개선
- 향후 과제: 실외 장시간 테스트, 방향 추정 정확도 검증

→ [프로젝트 상세 보기](/projects/raseyes/) · [관련 포스트 보기](/categories/raseyes/)

## 소개

이 블로그는 개발하면서 겪은 경험, 프로젝트 기록, 그리고 배운 것들을 정리하는 개인 기술 일지입니다.
