# Python 코딩 가이드

> 고등학생 및 교사를 위한 Python 미션 코드 작성 안내서
>
> `mobile_robot` 모듈을 사용하여 스마트 시티 미션을 수행하는 코드를 직접 Python으로 작성하는 방법을 안내합니다.

---

## 1. 코드 전체 구조

Python으로 미션 코드를 작성할 때, **아래 구조를 그대로 복사한 뒤 각 미션 함수 안에 여러분의 코드를 채워 넣으세요.**

```python
import mobile_robot
import time
import gc

# ============================================================
#                     미션 함수들
# ============================================================

def mission1():
  mobile_robot.run_command(mobile_robot.mission_start, "미션1 시작 실패", 1)

  # TODO: 여기에 미션1 코드를 작성하세요

  mobile_robot.run_command(mobile_robot.mission_complete, "미션1 완료 실패", 1, 0x01)

def mission2():
  mobile_robot.run_command(mobile_robot.mission_start, "미션2 시작 실패", 2)

  # TODO: 여기에 미션2 코드를 작성하세요

  mobile_robot.run_command(mobile_robot.mission_complete, "미션2 완료 실패", 2, 0x01)

def mission3():
  mobile_robot.run_command(mobile_robot.mission_start, "미션3 시작 실패", 3)

  # TODO: 여기에 미션3 코드를 작성하세요

  mobile_robot.run_command(mobile_robot.mission_complete, "미션3 완료 실패", 3, 0x01)

def mission4():
  mobile_robot.run_command(mobile_robot.mission_start, "미션4 시작 실패", 4)

  # TODO: 여기에 미션4 코드를 작성하세요

  mobile_robot.run_command(mobile_robot.mission_complete, "미션4 완료 실패", 4, 0x01)

# ============================================================
#                     시작 명령 대기
# ============================================================

def wait_for_start_command():
  print("시작 명령 대기 중...")
  while True:
    recv_type, _ = mobile_robot.receive_message(timeout_ms=100)
    if recv_type == mobile_robot.MsgType.START_CMD:
      mobile_robot.send_start_ack()
      print("시작 명령 수신! 미션 1~4 순차 실행 시작")
      return True
    elif recv_type == mobile_robot.MsgType.STOP_CMD:
      mobile_robot.send_stop_ack()
      print("정지 명령 수신 (대기 중)")
    time.sleep_ms(10)

# ============================================================
#                     미션 순차 실행
# ============================================================

def run_all_missions():
  mobile_robot.reset_mission_state()
  try:
    mission1()
    mission2()
    mission3()
    mission4()
    print("모든 미션 완료!")
    return True
  except mobile_robot.MissionError as e:
    print("미션 중단됨: {}".format(e.reason))
    return False

# ============================================================
#                     메인 실행
# ============================================================

# 로봇 연결
mobile_robot.reconnect(max_retries=5)

# 시작 명령 대기
wait_for_start_command()

# 미션 실행
result = run_all_missions()

# 가비지 컬렉션
gc.collect()
```

### 구조 설명

| 부분 | 역할 | 수정 여부 |
|------|------|-----------|
| `import` | 필요한 모듈 불러오기 | 수정하지 마세요 |
| `mission1()` ~ `mission4()` | 각 미션의 동작 코드 | **여기만 수정하세요** |
| `mission_start` / `mission_complete` | 서버에 미션 시작/완료를 알림 | 수정하지 마세요 |
| `wait_for_start_command()` | 키오스크에서 시작 버튼을 누를 때까지 대기 | 수정하지 마세요 |
| `run_all_missions()` | mission1~4를 순서대로 실행, 실패 시 중단 | 수정하지 마세요 |
| 메인 실행 | 로봇 연결 → 시작 대기 → 미션 실행 → 정리 | 수정하지 마세요 |

{% hint style="info" %}
**핵심 포인트**

- 미션은 항상 **1 → 2 → 3 → 4** 순서로 실행됩니다
- 어떤 미션에서든 실패(배터리 부족 등)하면 `MissionError`가 발생하여 나머지 미션을 건너뜁니다
- 여러분이 작성하는 코드는 각 `mission` 함수의 `# TODO` 부분에 들어갑니다
{% endhint %}

---

## 2. run\_command - 명령 실행 패턴

미션 함수 안에서 로봇에 명령을 내릴 때는 `run_command`를 사용합니다:

```python
mobile_robot.run_command(실행할_함수, "실패 시 메시지", 인자1, 인자2, ...)
```

**동작 방식:**

- 성공 → 다음 줄로 진행
- 실패 → `MissionError` 예외 발생 → 해당 미션 및 이후 미션 전체 중단
- 배터리 부족으로 서버가 거부 → 모터 정지 + LED 경고 + 미션 전체 중단

**따라서 여러분이 직접 에러 처리를 작성할 필요가 없습니다.** `run_command`가 알아서 처리합니다.

```python
# 예시: 2칸 전진 후 우회전 후 1칸 전진
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)
```

---

## 3. 함수 레퍼런스

`mobile_robot.` 접두사를 붙여서 호출합니다. (예: `mobile_robot.mission_move_forward(2)`)

### 3.1 이동

서버에 허가를 요청하고, 로봇이 동작을 완료할 때까지 자동 대기합니다.

#### `mission_move_forward(cells=1)`

라인트레이싱으로 지정한 칸 수만큼 전진합니다.

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `cells` | int | 이동할 칸 수 (1\~255, 기본값 1) |

| 반환값 | 설명 |
|--------|------|
| `(True, 에러코드)` | 성공 |
| `(False, 에러코드)` | 실패 (배터리 부족, 타임아웃 등) |

```python
# 1칸 전진
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

# 3칸 전진
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 3)
```

> 타임아웃: 기본 10초 + 칸당 10초 (예: 3칸 = 40초)

#### `mission_turn_left(n=1)`

90도씩 왼쪽으로 회전합니다.

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `n` | int | 회전 횟수 (기본값 1, 1회 = 90도) |

```python
# 좌회전 1번 (90도)
mobile_robot.run_command(mobile_robot.mission_turn_left, "회전 실패")

# 좌회전 2번 (180도 = U턴)
mobile_robot.run_command(mobile_robot.mission_turn_left, "회전 실패", 2)
```

#### `mission_turn_right(n=1)`

90도씩 오른쪽으로 회전합니다.

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `n` | int | 회전 횟수 (기본값 1, 1회 = 90도) |

```python
# 우회전 1번 (90도)
mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
```

---

### 3.2 신호등 감지

#### `detect_traffic_light()`

서버에 신호등 상태를 요청합니다.

| 반환값 | 설명 |
|--------|------|
| `(True, 신호등상태)` | 감지 성공 |
| `(False, 0xFF)` | 감지 실패 |

**신호등 상태값:**

| 값 | 의미 |
|----|------|
| `0` | 빨간불 (정지) |
| `1` | 노란불 (주의) |
| `2` | 초록불 (통과 가능) |
| `0xFF` | 감지 실패 |

```python
# 신호등이 초록불이 될 때까지 대기하는 패턴
while True:
  success, traffic_state = mobile_robot.detect_traffic_light()
  if not success:
    mobile_robot.handle_mission_abort("신호등 감지 실패")
    raise mobile_robot.MissionError("신호등 감지 실패")

  if traffic_state == 0:    # 빨간불
    time.sleep(0.3)         # 잠시 대기 후 다시 확인
  else:                     # 초록불 (또는 노란불)
    time.sleep(3)           # 신호 안정화 대기
    break                   # 루프 탈출 → 통과!

  time.sleep(0.01)          # CPU 과부하 방지
```

{% hint style="info" %}
`detect_traffic_light()`는 `run_command`를 쓰지 않고 직접 호출합니다. 반환값을 보고 판단해야 하기 때문입니다.
{% endhint %}

---

### 3.3 배터리 확인

#### `check_battery()`

서버에 배터리 잔량을 요청합니다.

| 반환값 | 설명 |
|--------|------|
| `(True, 잔량%)` | 확인 성공 (0\~100) |
| `(False, 0)` | 확인 실패 |

```python
# 배터리가 100%가 될 때까지 대기하는 패턴
while True:
  success, battery_level = mobile_robot.check_battery()
  if not success:
    mobile_robot.handle_mission_abort("배터리 확인 실패")
    raise mobile_robot.MissionError("배터리 확인 실패")

  if battery_level == 100:  # 충전 완료
    time.sleep(3)
    break
  else:                     # 아직 충전 중
    pass

  time.sleep(0.01)
```

---

### 3.4 충전

#### `charge_start()`

충전소에서 충전을 시작합니다. 서버에 허가를 요청합니다.

```python
mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")
```

#### `charge_stop()`

충전을 종료합니다.

```python
mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")
```

**충전 전체 흐름:**

```python
# 1. 충전 시작
mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")

# 2. 배터리 100%까지 대기
while True:
  success, battery_level = mobile_robot.check_battery()
  if not success:
    mobile_robot.handle_mission_abort("배터리 확인 실패")
    raise mobile_robot.MissionError("배터리 확인 실패")

  if battery_level == 100:
    time.sleep(3)
    break
  else:
    pass

  time.sleep(0.01)

# 3. 충전 종료
mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")
```

---

### 3.5 버튼 제어

#### `press_button_left()`

로봇의 왼쪽 버튼을 누릅니다.

```python
mobile_robot.run_command(mobile_robot.press_button_left, "왼쪽 버튼 실패")
```

#### `press_button_right()`

로봇의 오른쪽 버튼을 누릅니다.

```python
mobile_robot.run_command(mobile_robot.press_button_right, "오른쪽 버튼 실패")
```

> 버튼 명령의 타임아웃은 60초입니다. 서버 응답을 길게 기다립니다.

---

### 3.6 속도 제어

#### `set_speed_low()`

로봇의 이동 속도를 낮게 설정합니다. 이후 이동 명령들이 느리게 실행됩니다.

```python
mobile_robot.run_command(mobile_robot.set_speed_low, "속도 설정 실패")
```

---

### 3.7 에러 처리

#### `run_command(func, abort_reason, *args)`

명령을 실행하고, 실패 시 자동으로 미션을 중단합니다.

| 파라미터 | 설명 |
|---------|------|
| `func` | 실행할 함수 (예: `mobile_robot.mission_move_forward`) |
| `abort_reason` | 실패 시 출력할 메시지 |
| `*args` | 함수에 전달할 인자들 |

- 성공: 아무 일 없이 다음 줄로 진행
- 실패: `MissionError` 예외 발생 → 모터 정지 → LED 경고 → 서버에 중단 알림

#### `handle_mission_abort(reason)`

직접 미션을 중단할 때 사용합니다. 모터 정지 + LED 경고 + 서버 알림을 수행합니다.

```python
# 탐지 함수에서 실패 시 직접 중단 처리하는 패턴
success, value = mobile_robot.detect_traffic_light()
if not success:
  mobile_robot.handle_mission_abort("신호등 감지 실패")
  raise mobile_robot.MissionError("신호등 감지 실패")
```

#### `MissionError` 예외

미션 중단 시 발생하는 예외입니다. `run_all_missions()`에서 자동으로 잡아서 처리합니다.

| 속성 | 설명 |
|------|------|
| `reason` | 중단 사유 문자열 |
| `error_code` | 에러 코드 |

---

### 3.8 LED 제어

미션 코드에서 시각적 피드백이 필요할 때 사용할 수 있습니다.

#### `led_set(state)`

LED 상태를 설정합니다.

| state 값 | 상수명 | 동작 |
|----------|--------|------|
| `0` | `LED_OFF` | LED 끄기 |
| `1` | `LED_ON` | LED 켜기 |
| `2` | `LED_BLINK_SLOW` | 느리게 깜빡임 |
| `3` | `LED_BLINK_FAST` | 빠르게 깜빡임 |

```python
mobile_robot.led_set(mobile_robot.LED_ON)       # LED 켜기
mobile_robot.led_set(mobile_robot.LED_OFF)      # LED 끄기
```

#### `led_blink(on_ms, off_ms, count)`

LED 깜빡임 패턴을 설정합니다.

| 파라미터 | 설명 |
|---------|------|
| `on_ms` | 켜짐 시간 (밀리초) |
| `off_ms` | 꺼짐 시간 (밀리초) |
| `count` | 반복 횟수 |

```python
mobile_robot.led_blink(200, 200, 5)  # 0.2초 간격으로 5번 깜빡
```

---

### 3.9 저수준 모터 제어 (고급)

{% hint style="warning" %}
일반적으로 사용하지 않습니다. 미션 명령(`mission_move_forward` 등)은 서버 허가 + 라인트레이싱이 포함되어 있지만, 아래 함수들은 로봇 모터를 직접 제어하므로 라인트레이싱 없이 동작합니다.
{% endhint %}

| 함수 | 설명 |
|------|------|
| `move_forward(speed_us=500, steps=None)` | 직접 전진 (steps=None이면 연속 이동) |
| `move_backward(speed_us=500, steps=None)` | 직접 후진 |
| `turn_left(speed_us=800)` | 직접 좌회전 (연속 회전, stop() 필요) |
| `turn_right(speed_us=800)` | 직접 우회전 (연속 회전, stop() 필요) |
| `stop()` | 모터 즉시 정지 |
| `motor_move(motor, steps, speed_us)` | 특정 모터 스텝 이동 |
| `motor_stop(motor)` | 특정 모터 정지 |
| `motor_continuous(left_speed_us, right_speed_us)` | 양쪽 모터 연속 회전 |

**모터 선택 상수:**

| 상수 | 값 | 설명 |
|------|---|------|
| `MOTOR_LEFT` | 0 | 왼쪽 모터 |
| `MOTOR_RIGHT` | 1 | 오른쪽 모터 |
| `MOTOR_BOTH` | 2 | 양쪽 모터 |

> `speed_us`는 스텝 간 딜레이(마이크로초)입니다. **값이 작을수록 빠릅니다.**

---

## 4. 미션별 사용 가능 함수 요약

각 미션에서 사용할 수 있는 함수가 정해져 있습니다.

### 미션 1: 신호등 & 이동

| 함수 | 용도 |
|------|------|
| `mission_move_forward(cells)` | N칸 전진 (1\~4) |
| `mission_turn_right()` | 90도 우회전 |
| `mission_turn_left()` | 90도 좌회전 |
| `detect_traffic_light()` | 신호등 상태 확인 (0=빨강, 2=초록) |

**미션 1 예시 — 신호등 교차로 통과:**

```python
def mission1():
  mobile_robot.run_command(mobile_robot.mission_start, "미션1 시작 실패", 1)

  # 교차로까지 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 신호등 대기
  while True:
    success, traffic_state = mobile_robot.detect_traffic_light()
    if not success:
      mobile_robot.handle_mission_abort("신호등 감지 실패")
      raise mobile_robot.MissionError("신호등 감지 실패")

    if traffic_state == 0:    # 빨간불 → 대기
      time.sleep(0.3)
    else:                     # 초록불 → 통과
      time.sleep(3)
      break

    time.sleep(0.01)

  # 교차로 통과 후 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션1 완료 실패", 1, 0x01)
```

---

### 미션 2: 충전소 & 배터리

| 함수 | 용도 |
|------|------|
| `mission_move_forward(cells)` | N칸 전진 (1\~4) |
| `mission_turn_right()` | 90도 우회전 |
| `mission_turn_left()` | 90도 좌회전 |
| `charge_start()` | 충전 시작 |
| `charge_stop()` | 충전 종료 |
| `check_battery()` | 배터리 잔량 확인 (0\~100%) |

**미션 2 예시 — 충전소에서 충전:**

```python
def mission2():
  mobile_robot.run_command(mobile_robot.mission_start, "미션2 시작 실패", 2)

  # 충전소까지 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 충전 시작
  mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")

  # 배터리 100%까지 대기
  while True:
    success, battery_level = mobile_robot.check_battery()
    if not success:
      mobile_robot.handle_mission_abort("배터리 확인 실패")
      raise mobile_robot.MissionError("배터리 확인 실패")

    if battery_level == 100:
      time.sleep(3)
      break
    else:
      pass

    time.sleep(0.01)

  # 충전 종료 후 출발
  mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션2 완료 실패", 2, 0x01)
```

---

### 미션 3: 로봇 버튼 제어

| 함수 | 용도 |
|------|------|
| `mission_move_forward(cells)` | N칸 전진 (1\~4) |
| `mission_turn_right()` | 90도 우회전 |
| `mission_turn_left()` | 90도 좌회전 |
| `press_button_left()` | 로봇 왼쪽 버튼 누르기 |
| `press_button_right()` | 로봇 오른쪽 버튼 누르기 |

**미션 3 예시 — 버튼 조작:**

```python
def mission3():
  mobile_robot.run_command(mobile_robot.mission_start, "미션3 시작 실패", 3)

  # 제어 지점까지 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 3)
  mobile_robot.run_command(mobile_robot.mission_turn_left, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 왼쪽 버튼 누르기
  mobile_robot.run_command(mobile_robot.press_button_left, "왼쪽 버튼 실패")

  # 오른쪽 버튼 누르기
  mobile_robot.run_command(mobile_robot.press_button_right, "오른쪽 버튼 실패")

  # 다음 위치로 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션3 완료 실패", 3, 0x01)
```

---

### 미션 4: 속도 제어 & 반복

| 함수 | 용도 |
|------|------|
| `mission_move_forward()` | 1칸 전진 |
| `mission_turn_right()` | 90도 우회전 |
| `mission_turn_left()` | 90도 좌회전 |
| `set_speed_low()` | 속도 낮게 설정 |

> Python의 `for` 반복문을 활용할 수 있습니다.

**미션 4 예시 — 속도 조절 + 반복 이동:**

```python
def mission4():
  mobile_robot.run_command(mobile_robot.mission_start, "미션4 시작 실패", 4)

  # 속도 낮게 설정
  mobile_robot.run_command(mobile_robot.set_speed_low, "속도 설정 실패")

  # 2번 반복: 전진 → 우회전
  for i in range(2):
    mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")
    mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
    time.sleep(0.01)

  # 마지막 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")

  mobile_robot.run_command(mobile_robot.mission_complete, "미션4 완료 실패", 4, 0x01)
```

---

## 5. 에러 코드 정리

`run_command` 사용 시 내부에서 자동 처리되지만, 직접 함수를 호출할 때 참고합니다.

| 코드 | 이름 | 의미 |
|------|------|------|
| `0x00` | `ErrorCode.NONE` | 성공 |
| `0x01` | `ErrorCode.TIMEOUT` | 타임아웃 (응답 없음) |
| `0x02` | `ErrorCode.INVALID_MSG` | 잘못된 메시지 |
| `0x03` | `ErrorCode.UART_FAIL` | UART 통신 실패 |
| `0x06` | `ErrorCode.SERVER_FAIL` | 서버 거부 (배터리 부족 포함) |

---

## 6. 주의사항

{% hint style="danger" %}
**반드시 지켜야 할 규칙**

1. **들여쓰기**: 미션 함수 안의 코드는 **스페이스 2칸**으로 들여쓰기합니다.
2. **순서 실행**: 모든 명령은 위에서 아래로 순서대로 실행됩니다. 이전 명령이 완료되어야 다음 명령이 실행됩니다.
3. **`mission_start` / `mission_complete`**: 각 미션 함수의 맨 처음과 맨 끝에 반드시 있어야 합니다. 템플릿에 이미 포함되어 있으니 지우지 마세요.
4. **루프 안의 `time.sleep(0.01)`**: `while True` 루프 안에서 CPU 과부하를 방지하기 위해 반드시 넣어주세요.
{% endhint %}

{% hint style="info" %}
**알아두면 좋은 정보**

- **배터리 부족**: 서버가 배터리 부족을 감지하면 `run_command`가 자동으로 미션을 중단합니다. 직접 처리할 필요 없습니다.
- **타임아웃 기본값**:
  - 전진: 10초 + 칸당 10초 (예: 3칸 → 40초)
  - 회전: 회전당 10초
  - 버튼: 60초
  - 충전/신호등/배터리 확인: 5초
- **`detect_traffic_light()` / `check_battery()`**: 이 두 함수는 `run_command`를 쓰지 않고 직접 호출합니다. 반환값으로 판단해야 하기 때문입니다. 실패 시 `handle_mission_abort()` + `raise MissionError()`로 직접 중단 처리합니다.
{% endhint %}

---

## 7. 정답 코드

아래는 모든 미션을 정상 완료하는 전체 정답 코드입니다.
템플릿을 복사한 뒤, 각 `mission` 함수의 `# TODO` 부분을 아래 내용으로 채워 넣으면 됩니다.

### 미션별 정답 경로 요약

| 미션 | 동작 순서 |
|------|----------|
| **1** | 4칸 전진 → **신호등 대기**(빨간불: 대기, 초록불: 통과) → 3칸 전진 → 우회전 → 1칸 전진 |
| **2** | 1칸 전진 → 우회전 → 1칸 전진 → 우회전x2(U턴) → **충전 시작** → 배터리 100% 대기 → **충전 종료** → 1칸 전진 → 우회전 → 1칸 전진 |
| **3** | 1칸 전진 → 우회전 → 3칸 전진 → 우회전 → 1칸 전진 → **왼쪽 버튼** → 1칸 전진 → **오른쪽 버튼** → 우회전x2(U턴) → 1칸 전진 → 우회전 → 1칸 전진 |
| **4** | 1칸 전진 → 우회전 → 2번 반복(**속도 낮게** + 1칸 전진) |

### 전체 정답 코드 (복사용)

```python
import mobile_robot
import time
import gc

# ============================================================
#                     미션 함수들
# ============================================================

def mission1():
  mobile_robot.run_command(mobile_robot.mission_start, "미션1 시작 실패", 1)

  # 4칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 4)

  # 신호등 대기 (빨간불이면 대기, 초록불이면 통과)
  while True:
    success, traffic_state = mobile_robot.detect_traffic_light()
    if not success:
      mobile_robot.handle_mission_abort("신호등 감지 실패")
      raise mobile_robot.MissionError("신호등 감지 실패")

    if traffic_state == 0:    # 빨간불
      time.sleep(0.3)
      pass                    # 계속 탐지
    else:                     # 초록불
      time.sleep(0.5)
      break                   # 탐지 종료, 통과!

    time.sleep(0.3)

  # 3칸 전진 → 우회전 → 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 3)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션1 완료 실패", 1, 0x01)

def mission2():
  mobile_robot.run_command(mobile_robot.mission_start, "미션2 시작 실패", 2)

  # 1칸 전진 → 우회전 → 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 우회전 2번 (U턴)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")

  # 충전 시작
  mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")

  # 배터리 100%까지 대기
  while True:
    success, battery_state = mobile_robot.check_battery()
    if not success:
      mobile_robot.handle_mission_abort("배터리 감지 실패")
      raise mobile_robot.MissionError("배터리 감지 실패")

    if battery_state == 100:  # 충전 완료
      time.sleep(0.3)
      time.sleep(0.5)
      break                   # 탐지 종료
    else:                     # 아직 충전 중
      pass                    # 계속 탐지

    time.sleep(0.3)

  # 충전 종료
  mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")

  # 1칸 전진 → 우회전 → 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션2 완료 실패", 2, 0x01)

def mission3():
  mobile_robot.run_command(mobile_robot.mission_start, "미션3 시작 실패", 3)

  # 1칸 전진 → 우회전 → 3칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 3)

  # 우회전 → 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 왼쪽 버튼 누르기
  mobile_robot.run_command(mobile_robot.press_button_left, "왼쪽 버튼 실패")

  # 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 오른쪽 버튼 누르기
  mobile_robot.run_command(mobile_robot.press_button_right, "오른쪽 버튼 실패")

  # 우회전 2번 (U턴)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")

  # 1칸 전진 → 우회전 → 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션3 완료 실패", 3, 0x01)

def mission4():
  mobile_robot.run_command(mobile_robot.mission_start, "미션4 시작 실패", 4)

  # 1칸 전진 → 우회전
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")

  # 2번 반복: 속도 낮게 설정 → 1칸 전진
  for i in range(2):
    mobile_robot.run_command(mobile_robot.set_speed_low, "속도 설정 실패")
    mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")
    time.sleep(0.01)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션4 완료 실패", 4, 0x01)

# ============================================================
#                     시작 명령 대기
# ============================================================

def wait_for_start_command():
  print("시작 명령 대기 중...")
  while True:
    recv_type, _ = mobile_robot.receive_message(timeout_ms=100)
    if recv_type == mobile_robot.MsgType.START_CMD:
      mobile_robot.send_start_ack()
      print("시작 명령 수신! 미션 1~4 순차 실행 시작")
      return True
    elif recv_type == mobile_robot.MsgType.STOP_CMD:
      mobile_robot.send_stop_ack()
      print("정지 명령 수신 (대기 중)")
    time.sleep_ms(10)

# ============================================================
#                     미션 순차 실행
# ============================================================

def run_all_missions():
  mobile_robot.reset_mission_state()
  try:
    mission1()
    mission2()
    mission3()
    mission4()
    print("모든 미션 완료!")
    return True
  except mobile_robot.MissionError as e:
    print("미션 중단됨: {}".format(e.reason))
    return False

# ============================================================
#                     메인 실행
# ============================================================

# 로봇 연결
mobile_robot.reconnect(max_retries=5)

# 시작 명령 대기
wait_for_start_command()

# 미션 실행
result = run_all_missions()

# 가비지 컬렉션
gc.collect()
```
