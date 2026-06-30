# Level 2 프로젝트 진행 가이드

Level 2(Adaptive Navigation Agent)는 **카메라 기반 Perception + LLM Planning + 결정론적 Execution** 구조를 구현하는 과제다.

## 1. 목표

- 자연어 명령을 해석해 큐브를 색상별 destination pad로 운반한다.
- 소스 코드 수정 없이 시작 위치, 큐브 순서, 최종 평가 지시 변화에 대응한다.
- LLM은 **상위 의사결정**, deterministic code는 **실행**을 담당한다.

## 2. 제약

| 구분 | 내용 |
|---|---|
| 금지 | `scene_state`, entity id 기반 `go_to`, 기록/절대 좌표 기반 `go_to` , todo외 코드 변경|
| 허용 | 카메라 관찰값, 속도 제어, 머리 방향 제어, VLM, LLM decision loop |
| 결론 | 목표 탐색과 이동은 camera feedback 중심으로 수행한다. |

## 3. 환경 특징

| 고정 요소 | 매 실행 변경 요소 |
|---|---|
| 목적지 패드 위치 | 큐브 색상 등장 순서 |
| 색상 표지판 | 로봇 시작 위치 |
| 장애물 | 평가 시 자연어 지시 |
| 색상-패드 매칭 규칙 |  |

- 큐브는 항상 `A` 구역에서 하나씩 생성된다.
- 목적지는 `B red`, `C green`, `D blue`, `E yellow` 표지판/패드다.

## 4. 필수 LLM 역할

LLM은 저수준 제어값을 만들지 않는다. 다음만 결정한다.

- 다음 목표 선택
- 다음 high-level action 결정
- 실패 복구와 재시도 여부 결정
- 작업 종료 판단
- 자연어 지시 해석

필수 루프:

```text
관찰 -> 결정 -> 검증 -> 실행 -> 확인 -> 반복
```

## 5. LLM 출력 형식

예시:

```json
{
  "next_action": "search_cube",
  "target_color": "red",
  "reason": "red cube detected"
}
```

허용 action:

- `search_cube`
- `navigate_to_cube`
- `pick_cube`
- `search_pad`
- `navigate_to_pad`
- `place_cube`
- `recover`
- `skip_target`
- `stop`

모든 응답은 notebook의 `parse_agent_decision`으로 검증한다.

## 6. Memory 설계

LLM decision과 recovery에 최소한 다음 정보를 사용한다.

- `delivered_count`
- `delivery_limit`
- `priority_colors`
- `failed_attempts`
- `recent_outcomes`
- 현재 목표
- 현재 들고 있는 큐브
- 발견한 패드
- 탐색 완료 영역
- recover 횟수

Memory는 단순 로그가 아니라 **정책 변경 기준**이어야 한다.

## 7. 비공개 지시 대응

최종 평가에서는 자연어 명령이 바뀔 수 있다.

예시:

- 6개 중 4개만 운반
- 빨강/파랑 먼저 운반

대응 원칙:

- 하드코딩 금지
- prompt만으로 정책 변경 가능
- memory 기반으로 delivery limit, priority, skip/retry를 조정

## 8. 평가 방식과 배점

| 단계 | 조건 |
|---|---|
| 개발 | 랜덤 환경 자유 테스트 |
| 중간 평가 | 비공개 시작 위치, 비공개 큐브 순서 |
| 최종 평가 | 새로운 비공개 환경, 비공개 자연어 지시, 평가 중 코드 수정 불가 |

| 항목 | 배점 | 핵심 |
|---|---:|---|
| 작업 수행 | 40 | 올바른 분류, LLM 의사결정 효율 |
| Level 2 성공 | 30 | Level 2 제약 준수 |
| 비공개 지시 해결 | 20 | prompt와 memory만으로 대응 |
| 설계 및 발표 | 10 | 아키텍처, LLM 역할, 검증, 복구 전략 |

## 9. 구현 체크리스트

- [ ] `scene_state` 사용 안 함
- [ ] `go_to(entity id)` 사용 안 함
- [ ] 좌표 기반 `go_to` 사용 안 함
- [ ] 시작 위치 하드코딩 금지
- [ ] 큐브 순서 하드코딩 금지
- [ ] 카메라 기반 탐색 구현
- [ ] LLM decision loop 구현
- [ ] recover 전략 구현
- [ ] memory 시스템 구현
- [ ] prompt만으로 자연어 지시 변경 대응

## 10. 참조

- 현재 scaffold 상태: `analyzE_scaffold.md`
- 기술 세부사항 색인: `knowledge.md`
- 구현 레시피: `reference/implementation_recipes.md`
