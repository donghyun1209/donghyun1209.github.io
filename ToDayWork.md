# 2026-06-27 작업 일지

---

## Phase 5 · 시스템 최적화 및 안정화

### 5-1 · E2E Latency 프로파일링
| 파일 | 변경 내용 |
|------|-----------|
| `logs/logger.py` | `FIELDNAMES`에 `latency_ms` 컬럼 추가, `write_row()` 파라미터 추가 |
| `main.py` | `vision_ts` 기준 EMA 레이턴시 측정, 400ms 초과 시 경고 로그 + CSV 기록 |

### 5-2 · Thermal Graceful Degradation
| 파일 | 변경 내용 |
|------|-----------|
| `config.py` | `THERMAL_THROTTLE_TEMP_C = 80.0`, `THERMAL_THROTTLE_FPS = 5` 추가 |
| `main.py` | `_thermal_event` 추가, vision worker 스로틀 슬립 적용, 1초 주기 온도 체크 |

### 5-3 · 카메라 가림 감지
| 파일 | 변경 내용 |
|------|-----------|
| `config.py` | `CAMERA_OCCLUSION_CHANGE_THRESH = 3.0`, `CAMERA_OCCLUSION_FRAMES = 15`, `CAMERA_OCCLUSION_COOLDOWN_SEC = 5.0` 추가 |
| `main.py` | 연속 프레임 간 `np.mean(|curr - prev|)` 픽셀 변화량 측정, 15프레임 연속 < 3.0 시 HIGH 경보 + 5초 쿨다운 |

### 5-4 · 배터리 잔량 경고
| 파일 | 변경 내용 |
|------|-----------|
| `config.py` | `BATTERY_LOW_THRESHOLD_PCT = 20`, `BATTERY_CHECK_INTERVAL_SEC = 30.0`, `BATTERY_SYSFS_PATH` 추가 |
| `main.py` | `_read_battery_percent()` 신규 추가, 30초 주기 확인, 20% 미만 시 MID 경보 |

### 5-5 · 액티브 쿨러 PWM 제어
| 파일 | 변경 내용 |
|------|-----------|
| `scripts/pwm_fan_control.py` | 신규 생성 — `/sys/class/pwm/pwmchip0/pwm0/` sysfs 제어, 온도→듀티 선형 보간 (`<50°C→20%`, `50~70°C→20~80%`, `≥80°C→100%`), 5초 주기, SIGTERM/SIGINT 종료 |

> **비고**: 5V 상시 구동 팬을 GPIO 4/6번 핀에 직결로 하드웨어 해결. `pwm_fan_control.py`는 코드베이스에 보존하되 실제 미사용.

### 테스트
- `tests/test_phase5.py` 신규 생성 (20개 테스트)
- **최종 결과: 92 tests passing** (기존 72 + 신규 20)

---

## Phase 5 코드 리뷰 피드백 반영

> 출처: `feedback.txt` — 안정성·실시간성·예외처리 개선 8개 항목

### #1 · E2E 레이턴시 중복 누적 버그 (`main.py`)
- **문제**: `current_vision_ts`가 프레임 미수신 루프에서도 초기화되지 않아 EMA에 중복 누적
- **수정**: 레이턴시 계산 직후 `current_vision_ts = None` 리셋 → 신규 프레임 수신 시에만 갱신

### #2 · RKNN 클래스 라벨 불일치 (`vision/rknn_detector.py`)
- **문제**: `label=str(class_ids[i])` — 숫자 문자열(`"0"`, `"1"`) 반환, CPU 모드와 불일치
- **수정**: COCO 80개 클래스 테이블 `_COCO_CLASSES` 추가 → `label=_COCO_CLASSES[int(class_ids[i])]`

### #3 · 발열 스로틀링 히스테리시스 (`config.py`, `main.py`)
- **문제**: 80°C 경계에서 스로틀 on/off가 1초마다 반복되는 채터링 현상
- **수정**: `THERMAL_RECOVERY_TEMP_C = 75.0` 추가 — 진입 80°C 초과 / 복구 75°C 이하 (5°C 마진)

### #4 · ButtonHandler 연동 (`main.py`)
- **문제**: `sensor/button_handler.py` 구현체가 `main.py`에 연결되지 않아 물리 버튼 미작동
- **수정**: `_button_handler` / `_mute_active` 필드 추가, `_toggle_mute()` 콜백, `start()` / `stop()` 연동 (`use_hw` 모드에서만 활성화, gpiod 미설치 시 graceful 경고)

### #5 · PWM 초기화 순서 (`scripts/pwm_fan_control.py`)
- **문제**: 기존 `duty_cycle > period` 상태에서 `period` 쓰기 시 커널 `EINVAL` 오류
- **수정**: `period` 쓰기 전 `duty_cycle = 0` 먼저 기록

### #6 · SIGTERM Graceful Shutdown (`main.py`)
- **문제**: `KeyboardInterrupt`만 캐치 → systemd `SIGTERM` 수신 시 `finally` 블록 미실행, 리소스 미반환
- **수정**: `signal.SIGTERM` 핸들러 등록 (`_stop_event.set()`), `while True:` → `while not self._stop_event.is_set():`

### #7 · 오디오 출력 경합 방지 (`audio/beep_controller.py`, `main.py`)
- **문제**: 장애물 경보와 배터리 경고가 동시에 `play_alert()` 호출 시 ALSA 충돌 가능
- **수정**: `BeepController`에 `request_system_alert()` / `pop_system_alert()` 추가, 배터리 경고를 BeepController로 라우팅해 단일 경로로 직렬화. 음소거 플래그(`_mute_active`) 체크 추가

### #8 · NMSBoxes 반환 타입 안전성 (`vision/rknn_detector.py`)
- **문제**: OpenCV 버전에 따라 `NMSBoxes` 반환값이 list/tuple일 경우 `flatten()` AttributeError
- **수정**: `indices.flatten()` → `np.array(indices).flatten()`

### 테스트 결과
- **92 tests passing** (PC + Orange Pi 모두 통과, 회귀 없음)

---

### 오프라인 테스트 시 확인 필요 항목
1. **ButtonHandler** — gpiod 설치 여부, 버튼 누름 → 음소거 토글 로그 확인
2. **RKNN 라벨** — 탐지 결과가 `"person"` 등 문자열인지 확인 (숫자면 버그)
3. **발열 히스테리시스** — 80→75°C 구간에서 채터링 없는지 확인
4. **E2E latency CSV** — `latency_ms` 컬럼이 단조증가하지 않는지 확인
5. **SIGTERM cleanup** — `sudo systemctl stop raseyes` 후 `journalctl`에서 `"RasEyes 종료"` 확인
6. **오디오 경합** — 배터리 경고 시 경보음이 끊기지 않는지 귀로 확인
