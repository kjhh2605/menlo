# Implementation Recipes

워크숍 지식을 starter notebook의 함수에 연결하는 실전 레시피다.

## 1. Workbook 지식 -> Starter 적용 위치

| Workbook 지식 | Starter 적용 위치 |
|---|---|
| `session.get_vision("pov")` | `get_camera_frame(ctx)` |
| OpenCV HSV detection | `detect_color_blobs(jpeg)` 또는 custom perception 보강 |
| `set_head` scan | `scan_head(ctx)`, `visual_search(ctx, target_color)` |
| `set_velocity` short command | `move_velocity(ctx, vx, vy, wz, duration_s)` |
| `center_on_color` | `visual_navigate_to_target`의 정렬 단계 |
| `drive_toward_color` | `visual_navigate_to_target`의 접근 단계 |
| LLM JSON parser | `parse_agent_decision` |
| ReAct loop | `run_agent`의 observe-decide-act loop |
| `tool_log` | `AgentMemory.logs` |
| VLM `look` | `ask_vlm_about_frame`과 `build_signage_vlm_prompt` |

## 2. Cube 찾기

```text
1. scan_head(ctx)로 좌/중/우 detection 수집
2. target_color가 있으면 해당 색상만 필터
3. target_color가 없으면 memory.priority_colors, completed_colors를 고려해 후보 선택
4. blob_area가 가장 큰 후보를 active target으로 선택
5. 없으면 body를 조금 회전하고 재스캔
```

## 3. Cube 접근

```text
1. target_color detection 찾기
2. abs(angle) > 10이면 wz로 방향 보정
3. abs(angle) <= 10이면 짧게 전진
4. blob_area >= cube_arrival_area이면 pick 가능한 거리로 판단
5. 중간에 target을 잃으면 recover
```

## 4. Pad 찾기

```text
1. held_color로 목적지 sign 결정: red->B, green->C, blue->D, yellow->E
2. scan_head 또는 body rotation으로 sign/pad 탐색
3. VLM으로 visible sign letters/colors/left-center-right position 요청
4. 목적지 sign 방향으로 정렬
5. pad 근처까지 짧은 이동과 재관찰 반복
```

## 5. Place 직전 확인

```text
1. held_color가 존재해야 함
2. 목적지 sign 또는 pad color evidence가 있어야 함
3. 같은 color가 아닌 목적지에는 place 금지
4. 확신이 낮으면 recover/rescan
```

## 6. 피해야 할 패턴

- workbook의 `my_go_to_global`을 그대로 쓰기
- `scene_state`로 cube id, pad id, 좌표를 읽어 최종 logic에 넣기
- `go_to` 호출로 목적지 이동하기
- `pick_entity`를 정확한 cube id로 호출하기
- `place_entity`를 정확한 pad id로 호출하기
- 한 번의 긴 `set_velocity`로 이동을 끝내기
- LLM 응답을 JSON 검증 없이 바로 실행하기
- 실패 횟수 없이 같은 action을 무한 반복하기

## 7. 최종 구조 요약

Level 2 프로젝트에서는 `scene_state`, 정확한 entity id, `go_to` 기반 부분은 학습/검증용으로만 본다. 실제 구현은 다음 조합으로 완성한다.

```text
camera observation
+ color blob detection
+ set_head scan
+ short set_velocity control
+ VLM sign reading
+ LLM JSON decision
+ AgentMemory recovery
```
