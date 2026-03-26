# Python 코딩 가이드

> 고등학생 및 교사를 위한 Python 미션 코드 작성 안내서
>
> `mobile_robot` 모듈을 사용하여 스마트 시티 미션을 수행하는 코드를 직접 Python으로 작성하는 방법을 안내합니다.

***

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

### 어디를 수정해야 하나요?

| 부분                                   | 설명                         |   수정해도 되나요?  |
| ------------------------------------ | -------------------------- | :----------: |
| `import` 부분                          | 필요한 모듈을 불러옵니다              |       X      |
| `mission1()` \~ `mission4()` 함수      | 각 미션에서 로봇이 할 동작을 적는 곳      | **O (여기만!)** |
| `mission_start` / `mission_complete` | 서버에 "미션 시작/끝"을 알려줍니다       |       X      |
| `wait_for_start_command()`           | 키오스크에서 시작 버튼을 누를 때까지 기다립니다 |       X      |
| `run_all_missions()`                 | mission1→2→3→4 순서대로 실행합니다  |       X      |
| 메인 실행 부분                             | 로봇 연결 → 대기 → 실행 → 정리       |       X      |

{% hint style="info" %}
**핵심 포인트**

* 미션은 항상 **1 → 2 → 3 → 4** 순서로 실행됩니다
* 도중에 실패하면 나머지 미션은 자동으로 건너뜁니다
* 여러분이 코드를 적는 곳은 각 `mission` 함수의 `# TODO` 부분뿐입니다
{% endhint %}

***

## 2. 명령을 내리는 방법 — run\_command

로봇에게 "전진해!", "회전해!" 같은 명령을 내릴 때는 항상 `run_command`를 사용합니다.

```python
mobile_robot.run_command(실행할_명령, "실패했을 때 보여줄 메시지", 추가값1, 추가값2, ...)
```

* **성공하면** → 아무 문제 없이 다음 줄로 넘어갑니다
* **실패하면** → 자동으로 미션이 중단됩니다 (에러 처리를 직접 할 필요 없습니다)

```python
# 예시: 2칸 앞으로 → 오른쪽으로 회전 → 1칸 앞으로
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)
```

***

## 3. 사용할 수 있는 명령어 모음

***

### 3.1 전진 — mission\_move\_forward

로봇이 도로 위의 선을 따라 앞으로 이동합니다.

**설정값:**

| 이름      | 뜻          | 넣을 수 있는 값 | 안 넣으면 |
| ------- | ---------- | --------- | ----- |
| `cells` | 몇 칸 앞으로 갈지 | 1 \~ 255  | 1칸    |

**예시:**

```python
# 1칸 전진
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

# 3칸 전진
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 3)

# 숫자를 안 넣으면 1칸 전진
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")
```

***

### 3.2 좌회전 — mission\_turn\_left

로봇이 제자리에서 왼쪽으로 90도 회전합니다.

**설정값:**

| 이름  | 뜻                   | 넣을 수 있는 값 | 안 넣으면    |
| --- | ------------------- | --------- | -------- |
| `n` | 몇 번 회전할지 (1번 = 90도) | 1 이상      | 1번 (90도) |

**예시:**

```python
# 왼쪽으로 90도 회전
mobile_robot.run_command(mobile_robot.mission_turn_left, "회전 실패")

# 왼쪽으로 180도 회전 (U턴)
mobile_robot.run_command(mobile_robot.mission_turn_left, "회전 실패", 2)
```

***

### 3.3 우회전 — mission\_turn\_right

로봇이 제자리에서 오른쪽으로 90도 회전합니다.

**설정값:**

| 이름  | 뜻                   | 넣을 수 있는 값 | 안 넣으면    |
| --- | ------------------- | --------- | -------- |
| `n` | 몇 번 회전할지 (1번 = 90도) | 1 이상      | 1번 (90도) |

**예시:**

```python
# 오른쪽으로 90도 회전
mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")

# 오른쪽으로 180도 회전 (U턴)
mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패", 2)
```

***

### 3.4 신호등 확인 — detect\_traffic\_light

신호등이 무슨 색인지 확인합니다.

{% hint style="warning" %}
이 명령은 `run_command`를 쓰지 않고 **직접 호출**합니다. 신호등 색깔을 보고 판단해야 하기 때문입니다.
{% endhint %}

**신호등 색깔 값:**

| 값      | 무슨 색? | 뜻             |
| ------ | ----- | ------------- |
| `0`    | 빨간불   | 정지 — 기다려야 합니다 |
| `1`    | 노란불   | 주의            |
| `2`    | 초록불   | 통과 가능         |
| `0xFF` | —     | 감지 실패         |

**예시 — 초록불이 될 때까지 기다리기:**

```python
while True:
  success, traffic_state = mobile_robot.detect_traffic_light()
  if not success:
    mobile_robot.handle_mission_abort("신호등 감지 실패")
    raise mobile_robot.MissionError("신호등 감지 실패")

  if traffic_state == 0:    # 빨간불이면
    time.sleep(0.3)         #   잠깐 기다렸다가 다시 확인
  else:                     # 초록불이면
    time.sleep(3)           #   신호가 안정될 때까지 잠깐 기다린 후
    break                   #   루프 탈출! → 이제 출발

  time.sleep(0.01)          # CPU 과부하 방지 (반드시 넣어주세요)
```

***

### 3.5 배터리 확인 — check\_battery

로봇의 배터리가 몇 % 남았는지 확인합니다.

{% hint style="warning" %}
이 명령도 `run_command`를 쓰지 않고 **직접 호출**합니다.
{% endhint %}

**돌아오는 값:**

| 값             | 뜻                    |
| ------------- | -------------------- |
| `(True, 잔량%)` | 확인 성공 (0\~100 사이 숫자) |
| `(False, 0)`  | 확인 실패                |

**예시 — 배터리 100%가 될 때까지 기다리기:**

```python
while True:
  success, battery_level = mobile_robot.check_battery()
  if not success:
    mobile_robot.handle_mission_abort("배터리 확인 실패")
    raise mobile_robot.MissionError("배터리 확인 실패")

  if battery_level == 100:  # 100%면
    time.sleep(3)
    break                   #   충전 완료! 루프 탈출
  else:                     # 아직이면
    pass                    #   계속 확인

  time.sleep(0.01)          # CPU 과부하 방지
```

***

### 3.6 충전 시작 — charge\_start

충전소에 도착한 뒤 충전을 시작합니다.

**설정값:** 없음

**예시:**

```python
mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")
```

***

### 3.7 충전 종료 — charge\_stop

충전을 끝냅니다. 충전이 완료된 후에 호출합니다.

**설정값:** 없음

**예시:**

```python
mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")
```

***

### 3.8 충전 전체 흐름 (시작 → 대기 → 종료)

충전은 3단계로 이루어집니다:

```python
# 1단계: 충전 시작
mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")

# 2단계: 배터리가 100%가 될 때까지 기다리기
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

# 3단계: 충전 종료
mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")
```

***

### 3.9 왼쪽 버튼 누르기 — press\_button\_left

로봇의 왼쪽 버튼을 누릅니다.

**설정값:** 없음

**예시:**

```python
mobile_robot.run_command(mobile_robot.press_button_left, "왼쪽 버튼 실패")
```

***

### 3.10 오른쪽 버튼 누르기 — press\_button\_right

로봇의 오른쪽 버튼을 누릅니다.

**설정값:** 없음

**예시:**

```python
mobile_robot.run_command(mobile_robot.press_button_right, "오른쪽 버튼 실패")
```

{% hint style="info" %}
버튼 명령은 서버 응답을 오래 기다립니다 (최대 60초).
{% endhint %}

***

### 3.11 속도 낮추기 — set\_speed\_low

로봇의 이동 속도를 느리게 바꿉니다. 이후의 모든 이동이 천천히 실행됩니다.

**설정값:** 없음

**예시:**

```python
mobile_robot.run_command(mobile_robot.set_speed_low, "속도 설정 실패")

# 이제부터 느리게 이동합니다
mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
```

***

### 3.12 LED 제어 — led\_set

로봇의 LED를 켜거나 끄거나 깜빡이게 합니다.

**설정값:**

| 넣는 값 | 이름               | LED가 어떻게 되나요? |
| ---- | ---------------- | ------------- |
| `0`  | `LED_OFF`        | 꺼짐            |
| `1`  | `LED_ON`         | 켜짐            |
| `2`  | `LED_BLINK_SLOW` | 느리게 깜빡임       |
| `3`  | `LED_BLINK_FAST` | 빠르게 깜빡임       |

**예시:**

```python
# LED 켜기
mobile_robot.led_set(mobile_robot.LED_ON)

# LED 끄기
mobile_robot.led_set(mobile_robot.LED_OFF)

# LED 느리게 깜빡이기
mobile_robot.led_set(mobile_robot.LED_BLINK_SLOW)
```

***

### 3.13 LED 깜빡임 패턴 — led\_blink

LED를 원하는 패턴으로 깜빡이게 합니다.

**설정값:**

| 이름       | 뜻                   | 예시         |
| -------- | ------------------- | ---------- |
| `on_ms`  | LED가 켜져 있는 시간 (밀리초) | 200 = 0.2초 |
| `off_ms` | LED가 꺼져 있는 시간 (밀리초) | 200 = 0.2초 |
| `count`  | 몇 번 깜빡일지            | 5 = 5번     |

**예시:**

```python
# 0.2초 간격으로 5번 깜빡이기
mobile_robot.led_blink(200, 200, 5)

# 1초 간격으로 3번 깜빡이기
mobile_robot.led_blink(1000, 1000, 3)
```

***

### 3.14 저수준 모터 제어 (고급)

{% hint style="warning" %}
**일반적으로 사용하지 않습니다.** 위의 `mission_move_forward` 같은 명령은 서버 허가 + 라인트레이싱이 포함되어 있지만, 아래 함수들은 로봇 모터를 직접 제어하므로 선을 따라가지 않습니다.
{% endhint %}

| 함수                                        | 뜻                           |
| ----------------------------------------- | --------------------------- |
| `move_forward(speed_us=500, steps=None)`  | 직접 전진 (steps 없으면 계속 이동)     |
| `move_backward(speed_us=500, steps=None)` | 직접 후진                       |
| `turn_left(speed_us=800)`                 | 직접 좌회전 (계속 회전, `stop()` 필요) |
| `turn_right(speed_us=800)`                | 직접 우회전 (계속 회전, `stop()` 필요) |
| `stop()`                                  | 모터 즉시 정지                    |

> `speed_us`는 값이 **작을수록 빠릅니다.**

***

## 4. 미션별 예시 코드

각 미션에서 사용할 수 있는 함수가 정해져 있습니다.

***

### 미션 1: 신호등 & 이동

**사용 가능 함수:**

| 함수                          | 한 줄 설명 |
| --------------------------- | ------ |
| `mission_move_forward(칸 수)` | 앞으로 이동 |
| `mission_turn_right()`      | 오른쪽 회전 |
| `mission_turn_left()`       | 왼쪽 회전  |
| `detect_traffic_light()`    | 신호등 확인 |

**예시 — 신호등 교차로 통과:**

```python
def mission1():
  mobile_robot.run_command(mobile_robot.mission_start, "미션1 시작 실패", 1)

  # 교차로까지 1칸 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 초록불이 될 때까지 기다리기
  while True:
    success, traffic_state = mobile_robot.detect_traffic_light()
    if not success:
      mobile_robot.handle_mission_abort("신호등 감지 실패")
      raise mobile_robot.MissionError("신호등 감지 실패")

    if traffic_state == 0:    # 빨간불 → 기다림
      time.sleep(0.3)
    else:                     # 초록불 → 출발!
      time.sleep(3)
      break

    time.sleep(0.01)

  # 교차로 지나서 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션1 완료 실패", 1, 0x01)
```

***

### 미션 2: 충전소 & 배터리

**사용 가능 함수:**

| 함수                          | 한 줄 설명    |
| --------------------------- | --------- |
| `mission_move_forward(칸 수)` | 앞으로 이동    |
| `mission_turn_right()`      | 오른쪽 회전    |
| `mission_turn_left()`       | 왼쪽 회전     |
| `charge_start()`            | 충전 시작     |
| `charge_stop()`             | 충전 종료     |
| `check_battery()`           | 배터리 잔량 확인 |

**예시 — 충전소에서 충전하기:**

```python
def mission2():
  mobile_robot.run_command(mobile_robot.mission_start, "미션2 시작 실패", 2)

  # 충전소까지 이동
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 2)
  mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  # 충전 시작
  mobile_robot.run_command(mobile_robot.charge_start, "충전 시작 실패")

  # 100%가 될 때까지 기다리기
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

  # 충전 끝내고 출발
  mobile_robot.run_command(mobile_robot.charge_stop, "충전 종료 실패")
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패", 1)

  mobile_robot.run_command(mobile_robot.mission_complete, "미션2 완료 실패", 2, 0x01)
```

***

### 미션 3: 버튼 조작

**사용 가능 함수:**

| 함수                          | 한 줄 설명     |
| --------------------------- | ---------- |
| `mission_move_forward(칸 수)` | 앞으로 이동     |
| `mission_turn_right()`      | 오른쪽 회전     |
| `mission_turn_left()`       | 왼쪽 회전      |
| `press_button_left()`       | 왼쪽 버튼 누르기  |
| `press_button_right()`      | 오른쪽 버튼 누르기 |

**예시 — 버튼 누르기:**

```python
def mission3():
  mobile_robot.run_command(mobile_robot.mission_start, "미션3 시작 실패", 3)

  # 버튼이 있는 곳까지 이동
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

***

### 미션 4: 속도 제어 & 반복

**사용 가능 함수:**

| 함수                       | 한 줄 설명  |
| ------------------------ | ------- |
| `mission_move_forward()` | 앞으로 이동  |
| `mission_turn_right()`   | 오른쪽 회전  |
| `mission_turn_left()`    | 왼쪽 회전   |
| `set_speed_low()`        | 속도를 느리게 |

{% hint style="info" %}
Python의 `for` 반복문을 활용하면 같은 동작을 여러 번 반복할 수 있습니다!
{% endhint %}

**예시 — 속도 조절 + 반복 이동:**

```python
def mission4():
  mobile_robot.run_command(mobile_robot.mission_start, "미션4 시작 실패", 4)

  # 속도를 느리게 설정
  mobile_robot.run_command(mobile_robot.set_speed_low, "속도 설정 실패")

  # "전진 → 우회전"을 2번 반복
  for i in range(2):
    mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")
    mobile_robot.run_command(mobile_robot.mission_turn_right, "회전 실패")
    time.sleep(0.01)

  # 마지막 전진
  mobile_robot.run_command(mobile_robot.mission_move_forward, "전진 실패")

  mobile_robot.run_command(mobile_robot.mission_complete, "미션4 완료 실패", 4, 0x01)
```

***

## 5. 에러 코드 정리

`run_command`가 자동으로 처리해주지만, 참고로 알아두면 좋습니다.

| 코드     | 이름      | 무슨 뜻?             |
| ------ | ------- | ----------------- |
| `0x00` | 성공      | 명령이 잘 실행됨         |
| `0x01` | 타임아웃    | 응답이 너무 오래 안 옴     |
| `0x02` | 잘못된 메시지 | 보낸 메시지가 올바르지 않음   |
| `0x03` | 통신 실패   | UART 통신에 문제 발생    |
| `0x06` | 서버 거부   | 배터리 부족 등으로 서버가 거절 |

***

## 6. 꼭 지켜야 할 규칙

{% hint style="danger" %}
**중요한 규칙 4가지**

1. **들여쓰기는 스페이스 2칸** — 미션 함수 안의 코드는 반드시 2칸 들여쓰기
2. **순서대로 실행** — 위의 명령이 끝나야 아래 명령이 실행됩니다
3. **mission\_start / mission\_complete 지우지 마세요** — 각 미션 함수의 첫 줄과 마지막 줄에 이미 들어있습니다
4. **while 루프 안에 `time.sleep(0.01)` 넣기** — 안 넣으면 CPU에 과부하가 걸립니다
{% endhint %}

{% hint style="info" %}
**타임아웃 시간 안내**
{% endhint %}

| 명령         | 최대 대기 시간                   |
| ---------- | -------------------------- |
| 전진         | 10초 + 칸당 10초 (예: 3칸 = 40초) |
| 회전         | 회전당 10초                    |
| 버튼         | 60초                        |
| 충전/신호등/배터리 | 5초                         |

***

## 7. 정답 코드

### 미션별 정답 경로 요약

| 미션    | 동작 순서                                                                                         |
| ----- | --------------------------------------------------------------------------------------------- |
| **1** | 4칸 전진 → **신호등 대기** → 3칸 전진 → 우회전 → 1칸 전진                                                      |
| **2** | 1칸 전진 → 우회전 → 1칸 전진 → U턴 → **충전** → 1칸 전진 → 우회전 → 1칸 전진                                       |
| **3** | 1칸 전진 → 우회전 → 3칸 전진 → 우회전 → 1칸 전진 → **왼쪽 버튼** → 1칸 전진 → **오른쪽 버튼** → U턴 → 1칸 전진 → 우회전 → 1칸 전진 |
| **4** | 1칸 전진 → 우회전 → 2번 반복(**속도 낮게** + 1칸 전진)                                                        |

### 전체 정답 코

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

    if traffic_state == 0:
      time.sleep(0.3)
      pass
    else:
      time.sleep(0.5)
      break

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

    if battery_state == 100:
      time.sleep(0.3)
      time.sleep(0.5)
      break
    else:
      pass

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

  # 2번 반복: 속도 낮게 + 1칸 전진
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
