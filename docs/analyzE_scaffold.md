# Scaffold Analysis: Level 2 Autonomous Vision Agent

현재 `level_2_autonomous_vision_starter_ko.ipynb`는 과제 구조, schema, wrapper, agent loop를 제공하지만 6개 큐브를 실제로 분류 완료하는 solution은 아니다. 핵심 구현은 `STUDENT TODO` 함수에 남아 있다.

## 1. 과제와 목적지 규칙

Notebook의 과제 정의:

```python
TASK = "Find and sort the six cubes in the warehouse into their matching destination pads."
```

목적지 규칙:

```python
DESTINATION_SIGN_RULES = {
    "red": "B",
    "green": "C",
    "blue": "D",
    "yellow": "E",
}
```

- `A`는 conveyor/cube source 구역이며 목적지가 아니다.
- 목적지는 `B red`, `C green`, `D blue`, `E yellow` 표지판이다.

## 2. 핵심 파일/셀

| 위치 | 역할 |
|---|---|
| `level_2_autonomous_vision_starter_ko.ipynb` | 실제 starter scaffold. 과제 정의, memory/data schema, perception wrapper, robot action wrapper, agent loop 포함 |
| starter scaffold 코드 셀 | `ALLOWED_NEXT_ACTIONS`, dataclass, TODO 함수, 실행 루프 정의. 기존 과제 안내의 Cell 4 |
| robot 연결 셀 | `ctx` 연결. 기존 과제 안내의 Cell 6 |
| agent 실행 셀 | `memory = await run_agent(ctx)` 실행. 기존 과제 안내의 Cell 8 |
| `docs/Project_Guideline.md` | Level 2 제약, 평가 방식, LLM 역할, memory 설계 방향 |
| `docs/knowledge.md` | workbook에서 추출한 기술 지식 색인 |
| `.omx/` | 실행 환경 상태/로그. 프로젝트 구현 대상이 아님 |

## 3. 이미 제공되는 구성요소

### 3.1 Action schema

`ALLOWED_NEXT_ACTIONS`는 LLM이 반환할 수 있는 high-level action을 제한한다.

```python
ALLOWED_NEXT_ACTIONS = {
    "search_cube",
    "navigate_to_cube",
    "pick_cube",
    "search_pad",
    "navigate_to_pad",
    "place_cube",
    "recover",
    "skip_target",
    "stop",
}
```

`parse_agent_decision(text)`는 LLM 응답에서 JSON을 추출하고 `next_action` 허용 여부를 검증한다. 실제 LLM 호출을 추가해도 이 parser를 반드시 통과시킨다.

### 3.2 Agent data model

| Dataclass | 역할 | 주요 필드 |
|---|---|---|
| `AgentDecision` | LLM이 선택한 다음 행동 | `next_action`, `target_color`, `reason`, `recovery_strategy` |
| `AgentMemory` | cycle 사이 장기 상태 | `delivered_count`, `delivery_limit`, `priority_colors`, `held_color`, `active_color`, `stage`, `search_turns`, `failed_attempts`, `completed_colors`, `skipped_colors`, `logs` |
| `Observation` | LLM/action code에 전달되는 현재 관찰 | `robot_status`, `detections`, `note`, `vlm_summary` |
| `ScannedDetection` | head yaw/pitch와 color blob detection 저장 | `full_bearing_deg`는 image angle + head yaw 기반 body-relative bearing |

### 3.3 SDK wrapper

Level 2 제약에 맞는 wrapper가 이미 있다.

| Wrapper | 역할 |
|---|---|
| `get_robot_status(ctx)` | `ctx.state("robot_status")`로 상태 읽기 |
| `get_camera_frame(ctx)` | `ctx.get_vision("pov")`로 POV frame 획득 |
| `perceive(ctx)` | camera frame에 `detect_color_blobs(jpeg)` 적용 |
| `set_head(ctx, yaw, pitch)` | `ctx.invoke("set_head", ...)` |
| `move_velocity(ctx, vx, vy, wz, duration_s)` | `ctx.invoke("set_velocity", ...)`로 짧은 body-frame velocity command |
| `pick_nearest_cube(ctx)` | 가까운 cube pick. 정확한 cube id 사용은 금지. 시각적으로 의도한 cube 근처까지 이동 후 호출 |
| `place_nearest_zone(ctx)` | 가까운 zone place. matching pad 근처까지 시각적으로 이동 후 호출 |

### 3.4 VLM/scan helper

- `build_signage_vlm_prompt(held_color)`: 표지판 판독용 VLM prompt 생성.
- `ask_vlm_about_frame(ctx, prompt, api_key)`: 현재 POV frame을 `ask_vlm(...)`에 전달.
- 현재 scaffold에서는 VLM helper가 정의만 되어 있으며 `observe_world`/navigation에서 적극 사용되지는 않는다.
- `scan_head(ctx, yaws=(-0.8, 0.0, 0.8), pitch=0.15)`: 머리를 좌/중/우로 돌리며 detection 수집.

## 4. Agent loop

`run_agent(ctx, max_cycles=20)`의 cycle:

```text
observe_world(ctx, memory)
-> decide_next_action(TASK, observation, memory, last_result)
-> execute_decision(ctx, decision, observation, memory)
-> verify_outcome(ctx, decision, action_result)
-> update_memory(memory, observation, decision, verified)
```

구조는 Level 2 요구사항과 맞지만, 각 단계의 핵심 구현은 아직 최소 fallback 수준이다.

## 5. TODO/gap matrix

| 함수 | 현재 상태 | 부족한 점 |
|---|---|---|
| `decide_next_action` | context를 만들고 visible target이 없으면 `search_cube`, 있으면 가장 큰 blob 기준 `navigate_to_cube` 반환 | 실제 LLM 호출 없음, stage-aware decision 없음, holding 상태에서 pad 전환 없음, `pick/place/recover/stop` 판단 부족 |
| `observe_world` | `robot_status` 읽고 매번 `scan_head(ctx)` 실행, detection list 저장 | full scan/single frame 정책 없음, `vlm_summary` 미사용, cube/pad evidence 구분 없음, stage별 관찰 전략 없음 |
| `visual_search` | `scan_head(ctx)`만 호출하고 항상 `False` 반환 | target color filtering 없음, body rotation pattern 없음, cube/pad search 구분 없음, 찾은 target 기록 없음 |
| `visual_navigate_to_target` | 항상 `False` 반환 | closed-loop navigation 없음, `full_bearing_deg` steering 없음, blob area/position 도착 판정 없음, target loss/overshoot/obstacle 대응 없음 |
| `recover_motion` | 뒤로 이동 후 회전하고 result 반환 | 실패 종류별 전략 없음, 실패 횟수 제한/skip/retarget 없음, recovery 이후 재탐색과 memory update 약함 |
| `verify_outcome` | decision과 action_result를 그대로 묶어 반환 | pick/place 성공 검증 없음, camera evidence 재확인 없음, 다음 LLM call용 structured result 부족 |
| `update_memory` | visible count, decision, verified를 log에 기록 | `held_color`, `active_color`, `stage`, `delivered_count`, `completed_colors`, `failed_attempts`, priority/delivery limit 갱신 없음 |
| `execute_decision` | action별 wrapper routing | `visual_search`/`visual_navigate_to_target` 미구현 때문에 delivery 진행 불가, `skip_target`/`stop`/unknown no-op memory 정책 부족 |

## 6. 가능한 것과 불완전한 것

이미 가능한 것:

- 로봇 context 연결
- POV camera frame 획득
- color blob detection
- head scan
- velocity command 호출
- nearest cube pick / nearest zone place 호출
- observe-decide-act loop 실행
- 최소 log 기록

아직 불가능하거나 불완전한 것:

- 큐브 6개 안정 운반
- 색상별 목적지까지 vision-only navigation
- LLM 기반 stage-aware decision
- pick/place 성공 검증
- 실패 복구와 반복 실패 방지
- 비공개 자연어 지시에 대한 prompt/memory 기반 대응

## 7. 구현 우선순위

1. `decide_next_action`: 실제 LLM 호출과 JSON-only schema 강제.
2. `update_memory`: `stage`, `active_color`, `held_color`, `delivered_count`, `failed_attempts` 갱신.
3. `visual_search`: 목표 색상과 stage에 맞는 scan/body rotation search.
4. `visual_navigate_to_target`: `observe -> short move -> observe -> correct/stop` loop.
5. `verify_outcome`: pick/place/navigate 결과를 camera evidence와 SDK result로 확인.
6. `recover_motion`: 실패 종류와 횟수에 따라 backoff, rescan, skip, retarget 선택.
7. `run_agent`: `delivery_limit`, `delivered_count`, `completed_colors`, `stop` decision 기반 종료.

## 8. Level 2 제약 관점 주의점

반드시 피할 것:

- `scene_state` 사용
- 정확한 cube/pad entity id 사용
- 좌표 기반 `go_to` 사용
- 시작 위치, 큐브 순서, 평가 환경 좌표 하드코딩
- LLM 없이 deterministic fallback만으로 전체 과제를 해결하려는 구현

허용 방향:

- camera frame 기반 perception
- `set_head` scan
- `set_velocity` 기반 짧은 closed-loop motion
- VLM을 통한 표지판/목적지 확인
- LLM의 high-level action decision
- memory 기반 복구와 재시도 제한

## 9. 결론

Scaffold는 action schema, memory 구조, wrapper, agent loop를 제공하므로 구현 위치는 명확하다. 실제 성공 여부는 `decide_next_action`, `visual_search`, `visual_navigate_to_target`, `verify_outcome`, `update_memory`를 완성해 다음 구조를 실제 동작으로 연결하는 데 달려 있다.

```text
camera 기반 perception + LLM planning + deterministic execution
```
