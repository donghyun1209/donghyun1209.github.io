# 1주차 노트 (10/18~10/24) · 몸풀기와 환경 세팅

## 한 일
- [x] `.venv` 가상환경 + torch(CPU), ultralytics, onnx, onnxruntime 설치
- [x] YOLOv8n → ONNX export (`export_onnx.py`)
- [x] Netron으로 입력·출력 shape 확인

## export 코드 (`export_onnx.py`)
```python
from ultralytics import YOLO
yolo = YOLO("yolov8n.pt")               # 없으면 자동 다운로드
yolo.export(format="onnx", imgsz=640)   # 입력 크기 고정 → 양자화·RKNN 변환 때 덜 막힘
```

## export 로그에서 본 것
| 항목 | 값 | 메모 |
|---|---|---|
| 입력 | `(1, 3, 640, 640)` BCHW | 배치 1, RGB 3채널, 640×640 |
| 출력 | `(1, 84, 8400)` | 아래 해석 |
| opset | 18 | RKNN 등 변환 도구는 지원 opset 범위가 있음, 막히면 먼저 확인 |
| 크기 | `.pt` 6.2MB → `.onnx` 12.3MB | `.pt`는 FP16 저장, ONNX는 FP32 → 2배. 추후 INT8로 얼마나 줄지 비교 |

## 출력 `(1, 84, 8400)` 해석
- **84 = 4 + 80**
  - 4: 박스 `cx, cy, w, h` (640 입력 기준 픽셀)
  - 80: COCO 클래스 점수 (0~79), 0~1 값
  - YOLOv5는 objectness 1개가 더 있어서 85였으나, v8에서는 없는것으로 알고 있다.
- **8400 = 80² + 40² + 20²** = 박스 후보 수
  - 640 ÷ 8/16/32 = 80/40/20 → 격자 80×80, 40×40, 20×20
  - 격자 칸마다 후보 1개 → 6400 + 1600 + 400
  - 세 격자 결과를 **이어 붙임(Concat)**

| 격자 | stride | 잘 잡는 것 |
|---|---|---|
| 80×80 | 8 | 작은 물체 |
| 40×40 | 16 | 중간 물체 |
| 20×20 | 32 | 큰 물체 |

- 후보 하나 꺼내기: `out[0, :, i]` → 84개 값 (`:` = 그 축 전부)

## 용어 정리
- **NMS (Non-Maximum Suppression)**: 추론 *후*, 겹친 박스(IoU 기준) 중 점수 1등만 남기고 나머지 제거 
- **양자화**: 배포 *전*, 가중치 FP32 → INT8로 모델을 작고 빠르게 

## RasEyes 연결
- RasEyes RKNN 변환도 `.pt → ONNX → rknn-toolkit2 → .rknn` 흐름. 이번 주는 그 중간 단계(ONNX)를 직접 만들고 복습
- RasEyes 후처리의 출력 3개 = 위 격자 3개 구조

