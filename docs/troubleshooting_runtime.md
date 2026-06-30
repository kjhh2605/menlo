# Runtime Troubleshooting Notes: Level 2 Vision Agent

이 문서는 `level_2_autonomous_vision_starter_ko.ipynb`를 VSCode/Jupyter에서 실제 실행하며 발견한 문제와 대응을 기록한다.

## 1. `NameError: run_agent is not defined`

### 증상

```python
memory = await run_agent(ctx)
```

실행 시:

```text
NameError: name 'run_agent' is not defined
```

### 원인

Jupyter/VSCode Notebook에서는 함수 정의가 들어있는 scaffold 코드 셀을 먼저 실행해야 한다.  
`run_agent`, `decide_next_action`, `visual_navigate_to_target` 등은 긴 scaffold 코드 셀 안에서 정의된다.

### 조치

1. `"""Level 2 프로젝트 스타터입니다...`로 시작하는 긴 scaffold 코드 셀 실행
2. robot 연결 셀 실행 또는 기존 `ctx` 확인
3. 아래 체크 실행

```python
print("run_agent" in globals())
print("ctx" in globals())
```

둘 다 `True`여야 한다.

---

## 2. `navigate_to_cube`만 반복되고 로봇이 거의 안 움직임

### 증상

```text
[Level 2] Cycle 1
LLM decision: AgentDecision(next_action='navigate_to_cube', target_color='blue', ...)

[Level 2] Cycle 2
LLM decision: AgentDecision(next_action='navigate_to_cube', target_color='blue', ...)
...
```

로봇이 눈에 띄게 이동하지 않고 같은 action이 반복된다.

### 원인

초기 `visual_navigate_to_target()`가 접근 단계에서도 매번 `scan_head()`를 넓게 수행했다.

실제 흐름은 다음에 가까웠다.

```text
머리 왼쪽 보기 -> 가운데 보기 -> 오른쪽 보기 -> 아주 짧게 이동 -> 다시 scan
```

따라서 실행 시간 대부분이 head scan에 쓰이고, 전진 이동이 누적되지 않았다. 또한 navigation 실패 후 memory가 다시 같은 visible blob을 선택해 `navigate_to_cube`가 반복됐다.

### 조치

접근 단계는 wide scan이 아니라 정면 camera feedback 중심으로 변경한다.

- `set_head(yaw=0.0, pitch=0.15)` 고정
- `perceive(ctx)`로 정면 frame 반복 관찰
- `angle_deg`로 회전
- `blob_area`로 접근 완료 판정
- debug 출력 추가

예상 로그:

```text
navigate: blue angle=-5.4 area=38632
Action result: {'action': 'navigate_to_cube', 'reached': True}
Verified: {'ok': True, ...}
```

---

## 3. `search_cube` 직후 바로 `pick_cube`를 시도함

### 증상

```text
Cycle 1
search_cube found True

Cycle 2
LLM decision: pick_cube green
Action result: {'status': 'blocked_precondition', 'reason': 'not visually confirmed near selected cube'}
```

### 원인

`search_cube` 성공은 “큐브를 봤다”는 뜻이지 “큐브 근처까지 갔다”는 뜻이 아니다.  
그런데 memory stage가 잘못 해석되어 LLM이 `navigate_to_cube` 없이 `pick_cube`를 선택할 수 있었다.

### 조치

- `search_cube` 성공 후에도 near/pick-ready stage로 바꾸지 않는다.
- `memory.stage != "ready_to_pick"`이면 LLM의 `pick_cube` 결정을 무효화하고 fallback이 `navigate_to_cube`를 고르게 한다.

정상 흐름:

```text
search_cube -> navigate_to_cube -> pick_cube
```

---

## 4. `target_color`와 실제 집은 큐브 색상이 다름

### 증상

로그상으로는 blue를 집었다고 나오지만 실제 viewer에서는 green cube를 들고 있다.

```text
LLM decision: pick_cube target_color='blue'
Action result: {'status': 'done'}
...
search_pad target_color='blue'
```

### 원인

`pick_entity` wrapper는 정확한 색상 cube를 집는 API가 아니라, 가까운 cube를 집는다.  
따라서 `decision.target_color`는 “의도한 색”일 뿐, 실제 held color가 아닐 수 있다.

기존 memory update가 다음처럼 동작하면 오판한다.

```python
memory.held_color = decision.target_color
```

### 조치

`pick_cube` 직후 post camera scan에서 가장 큰 color blob을 `held_color_guess`로 기록하고, memory의 `held_color`를 camera evidence 기준으로 보정한다.

예상 로그:

```text
Verified: {..., 'held_color_guess': 'green'}
```

보정 기록:

```python
{"held_color_corrected_from_camera": {"intended": "blue", "actual": "green"}}
```

이후 pad 탐색은 실제 held color 기준으로 진행한다.

---

## 5. `place_cube`가 `done`인데 matching pad에 놓았는지 불명확함

### 증상

```text
navigate_to_pad target_color='green'
Action result: {'action': 'navigate_to_pad', 'reached': True}
place_cube
Action result: {'status': 'done'}
```

하지만 viewer에서는 destination pad까지 이동한 것이 보이지 않거나, 어디에 놓았는지 불명확하다.

### 원인

큐브를 들고 있으면 카메라 하단/중앙에 held cube 색상 blob이 크게 잡힌다.  
pad navigation이 color blob만 보고 있으면 held cube 자체를 matching pad/sign으로 착각할 수 있다.

### 조치

`navigate_to_cube`와 `navigate_to_pad`를 분리한다.

- cube 접근: 정면 blob area/angle 기반
- pad 접근: held cube로 보이는 blob을 후보에서 제외

`visual_navigate_to_pad()`에서 다음 정보를 출력한다.

```text
pad-nav: green bearing=+17.9 area=12303 cy=624 held_like=True
```

---

## 6. `pad-nav`에서 `cy=624` 같은 하단 blob을 계속 추적함

### 증상

```text
pad-nav: green bearing=+17.9 area=12303 cy=624 held_like=False
pad-nav: green bearing=+17.9 area=12265 cy=624 held_like=False
...
```

### 원인

`cy=624`는 화면 거의 아래쪽이다. 목적지 sign/pad라기보다 들고 있는 큐브, 발밑, 또는 로봇 가까운 하단 blob일 가능성이 높다.

초기 held-like 판정은 `area > 45000 and cy > 180`처럼 너무 큰 blob만 제외해서, 면적은 중간이지만 화면 하단인 blob을 pad 후보로 착각했다.

### 조치

held-like 판정을 강화한다.

```python
held_like = (area > 45000 and cy > 180) or cy > 560
```

그리고 pad 후보 filtering에서도 `cy > 560`을 제외한다.

좋은 로그 예:

```text
pad-nav: green ... cy=624 held_like=True
pad-nav: green ... cy=200 held_like=False
```

---

## 7. LiveKit `Ack received for unexpected RPC request` warning

### 증상

```text
livekit::room::participant::local_participant - Ack received for unexpected RPC request: ...
livekit::room::participant::local_participant - Response received for unexpected RPC request: ...
```

### 판단

현재 관찰된 실행에서는 이 warning 자체가 agent loop 실패의 직접 원인은 아니었다.  
`set_head`, `set_velocity`, `pick_entity`, `place_entity` 호출은 계속 진행됐다.

### 조치

우선 무시 가능. 단, command가 실제로 실행되지 않는 증상이 함께 있으면 `Action result`, `Verified`, robot viewer 움직임을 같이 확인한다.

---

## Debug 로그 해석 기준

### cube 접근

```text
navigate: blue angle=-5.4 area=38632
```

- `angle`: target blob의 좌우 위치
- `area`: 가까워질수록 커짐
- area가 threshold 이상이면 `reached=True`

### pad 접근

```text
pad-nav: green bearing=+17.9 area=12303 cy=624 held_like=True
```

- `bearing`: body 기준 방향
- `area`: 후보 blob 크기
- `cy`: 화면 세로 위치. 너무 크면 화면 아래쪽
- `held_like=True`: 들고 있는 큐브/하단 blob으로 보고 목적지 후보에서 제외

## 남은 위험

- **위험**: color blob만으로 pad/sign과 held cube를 완전히 구분하기 어렵다.
- **위험**: `cy` threshold와 `area` threshold는 camera 해상도/시점에 따라 runtime tuning이 필요할 수 있다.
- **위험**: 최종 안정화를 위해서는 pad 접근 단계에 VLM sign confirmation을 추가하는 것이 가장 안전하다.
