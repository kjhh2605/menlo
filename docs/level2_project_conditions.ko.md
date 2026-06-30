# Level 2 프로젝트 조건

이 문서는 Level 2 프로젝트 완성에 종속되는 조건과 제약만 모은다. 코드 내부 구조와 구현 힌트는 [level2_background_knowledge.ko.md](level2_background_knowledge.ko.md)를 본다.

## 제출/작업 표면

- 최종 제출물은 `notebooks/project/ko/level_2_autonomous_vision_starter_ko.ipynb` 하나다.
- 새 `.py` 구현 파일이나 local test file을 만들지 않는다.
- notebook 안에서는 주석과 안내가 표시한 TODO/학생 구현 구역만 수정한다.
- setup, robot 연결, 실행 cell, 공식 안내 markdown은 starter 흐름 보존을 위해 임의로 바꾸지 않는다.

## Level 2 목표 구조

구조는 항상 다음 역할 분리를 지킨다.

```text
camera perception -> LLM high-level decision -> deterministic execution
```

LLM은 high-level task supervisor다. Low-level perception, navigation, manipulation, validation, safety는 deterministic code가 처리한다.

## 평가/실행 조건

- Starter run path는 기본적으로 10분 scored simulation을 실행한다.
- Level 2는 successful delivery 1개당 30점이다.
- cube 개수 상한과 100점 상한은 없다.
- `MENLO_API_KEY`와 `TOKAMAK_API_KEY`가 필요하다.

## Hard bans

최종 Level 2 agent logic에 아래 방식을 넣지 않는다.

- `scene_state`
- coordinate/global `go_to`
- 기록된 절대 좌표 이동
- 정확한 cube/pad entity id 기반 이동, pick, place shortcut
- 시작 위치, cube 순서, 평가 환경 shortcut 하드코딩
- LLM이 속도값/좌표/low-level command를 직접 출력하는 구조

## Allowed control surface

- POV camera frame
- `detect_color_blobs`
- `set_head` scan
- 짧은 `set_velocity` closed-loop movement
- optional VLM sign/pad 확인
- LLM JSON action selection
- `AgentMemory` 기반 stage, retry, recovery 관리

`pick_entity`/`place_entity` wrapper는 camera 기반으로 의도한 cube 또는 matching pad 근처까지 이동한 뒤에만 호출한다.

## LLM contract

- LLM 출력은 JSON이다.
- `parse_agent_decision`으로 검증한다.
- `next_action`은 starter의 `ALLOWED_NEXT_ACTIONS`만 허용한다.
- invalid JSON/action은 안전하게 `recover` 또는 `search_cube` 계열 fallback으로 처리한다.
- LLM은 velocity, coordinate, entity id, SDK command를 직접 제어하지 않는다.

허용 action:

```text
search_cube
navigate_to_cube
pick_cube
search_pad
navigate_to_pad
place_cube
recover
skip_target
stop
```

## Memory contract

`AgentMemory`는 최소한 다음 필드를 일관되게 갱신한다.

- `stage`
- `active_color`
- `held_color`
- `delivered_count`
- `completed_colors`
- `failed_attempts`
- `logs`

반복 실패는 `failed_attempts` 기반으로 recover/skip/retarget 처리한다.

## 검증 조건

변경 후 가능한 범위에서 확인한다.

- Notebook JSON parse 성공.
- Notebook code cell 안의 금지 API/shortcut 미사용.
- `ALLOWED_NEXT_ACTIONS`와 LLM schema 일치.
- Runtime 가능 시 `memory = await run_agent(ctx)`로 10분 scored simulation 확인.

Runtime/browser 검증이 불가능하면 “실제 delivery 성공”을 주장하지 말고 **위험**으로 표시한다.

