# Agent Rules

저장소 전체 규칙. 상세 내용은 중복하지 않고 문서로 참조한다.

## Read first

- 문서 지도: `docs/README.md`
- 과제/평가 기준: `docs/Project_Guideline.md`
- 현재 scaffold/TODO: `docs/analyzE_scaffold.md`
- 기술 참조: `docs/knowledge.md`

## Goal

`level_2_autonomous_vision_starter_ko.ipynb`에서 **Level 2 Autonomous Vision Agent**를 완성한다.

구조는 항상 다음 역할 분리를 지킨다.

```text
camera perception -> LLM high-level decision -> deterministic execution
```

## Hard bans

최종 agent logic에 아래 방식을 넣지 않는다.

- `scene_state`
- 좌표 기반 `go_to` 또는 기록된 절대 좌표 이동
- 정확한 cube/pad entity id 기반 이동, pick, place
- 시작 위치/큐브 순서/평가 환경 shortcut 하드코딩

## Allowed control surface

- POV camera frame
- `detect_color_blobs`
- `set_head` scan
- 짧은 `set_velocity` closed-loop movement
- VLM sign/pad 확인
- LLM JSON action selection
- `AgentMemory` 기반 stage, retry, recovery 관리

`pick_entity`/`place_entity`는 카메라 기반으로 의도한 cube 또는 matching pad 근처까지 이동한 뒤에만 호출한다.

## LLM contract

- LLM은 속도값을 직접 제어하지 않는다.
- LLM 출력은 JSON이며 `parse_agent_decision`으로 검증한다.
- `next_action`은 notebook의 `ALLOWED_NEXT_ACTIONS`만 허용한다.
- invalid JSON/action은 안전하게 `recover` 또는 `search_cube`로 fallback한다.

## Memory contract

`AgentMemory`는 최소한 `stage`, `active_color`, `held_color`, `delivered_count`, `completed_colors`, `failed_attempts`, `logs`를 일관되게 갱신한다.

반복 실패는 `failed_attempts` 기반으로 recover/skip/retarget 처리한다.

## Implementation priority

1. `decide_next_action`
2. `update_memory`
3. `visual_search`
4. `visual_navigate_to_target`
5. `verify_outcome`
6. `recover_motion`
7. `run_agent` 종료 조건

## Verification

변경 후 확인한다.

- Notebook JSON parse 성공
- 금지 API/shortcut 미사용
- `ALLOWED_NEXT_ACTIONS`와 LLM schema 일치
- 실행 전 scaffold 코드 셀 재실행 → robot 연결 셀 → `memory = await run_agent(ctx)`

## Writing style

- 한국어 기본, 코드 식별자는 원문 유지
- “이미 동작함”과 “TODO/fallback” 구분
- 평가 제약 충돌 가능성은 **위험**으로 표시
