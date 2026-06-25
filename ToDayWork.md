## Phase 4-D 진행 현황 (2026-06-25)

### 완료
- [x] Orange Pi 5에서 의존성 설치 — sounddevice, gpiod, VL53L1X pip 설치 완료
- [x] 카메라 / ToF 단품 테스트 — CSICameraHAL PASS, VL53L1XHAL PASS
- [x] `RASEYES_HW=1 python main.py` 통합 테스트 — ToF 실측으로 MID/HIGH 경보 정상 동작 확인
- [x] `scripts/test_device.py` 작성 — 단품 테스트 자동화 스크립트
- [x] `scripts/bench_rknn.py` 작성 — RKNN 추론 속도 측정 스크립트

### 오늘 발견 및 수정한 버그
- `sensor/vl53l1x_hal.py`:
  - `VL53L1X.VL53L1X._TOF_LIBRARY` → `VL53L1X._TOF_LIBRARY` (모듈 레벨 변수)
  - `i2c_port=` → `i2c_bus=` (설치된 라이브러리 파라미터명)
  - `start_ranging(3)` LONG 모드 → `start_ranging(2)` MEDIUM 모드
    - LONG 모드는 최소 140ms 타이밍 버짓 필요 / 50ms 설정 시 측정 주기 ~1s → 데이터 만료 경고 flooding
    - MEDIUM(2)은 최대 3m, 33ms+ 버짓 지원 → 50ms 설정으로 ~10 Hz 안정 동작
- `main.py`:
  - rknnlite2/ultralytics/PortAudio 미설치 시 `start()` 크래시 대신 graceful fallback 추가
  - `_build_vision()`에서 `.rknn` 파일 존재 여부를 factory 단계에서 확인 → 없으면 MockVision fallback

### 추가 완료 (2026-06-25 오후)
- [x] systemd 서비스 등록 — `enabled`, `active (running)` 확인
- [x] 부팅 시간 — kernel 3.8s + userspace 7.2s = **11초** (KPI 45초 통과)
- [x] JackAudioHAL 정상 동작 확인 — sounddevice가 시스템 PortAudio 자동 탐지 (apt 설치 불필요)
- [x] 센서 데이터 만료 경고 해소 — MEDIUM 모드 적용 후 경고 0건
- [x] `.github/workflows/build_rknn.yml` 작성 — GitHub Actions(ubuntu x86_64)에서 자동 변환
  - torch CPU 별도 설치 → PyPI 의존성 충돌 수정 2회

### 참고 — 하드웨어 / 운영 현황
- CSI 카메라 FPS: ~10 FPS (KPI 15 FPS 미달, `/dev/video11` V4L2 드라이버 한계)
- 현재 서비스 상태: MockVision + VL53L1XHAL + JackAudioHAL (ToF 실측 + 이어폰 경보 정상)
- rknnlite2 설치됨 — `yolov8n.rknn` 파일만 전송하면 즉시 NPU 추론 모드로 전환
