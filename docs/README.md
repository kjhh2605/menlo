# Documentation Map for AI Agents

이 디렉터리는 Level 2 Autonomous Vision Agent 구현에 필요한 지식을 **빠르게 찾고, 세부사항은 참조 문서로 이동**하도록 정리한다.

## 읽는 순서

1. `../agents.md` — 저장소 전체 작업 규칙과 금지사항.
2. `Project_Guideline.md` — Level 2 요구사항, 평가 기준, 체크리스트.
3. `analyzE_scaffold.md` — 현재 notebook scaffold 상태와 TODO 위치.
4. `knowledge.md` — 워크숍에서 추출한 기술 지식의 색인.

## 파일 역할

| 파일 | 역할 |
|---|---|
| `Project_Guideline.md` | 목표, 제약, LLM 역할, 평가 방식 요약 |
| `analyzE_scaffold.md` | starter notebook의 제공 기능과 구현 공백 분석 |
| `knowledge.md` | 기술 지식 색인. 긴 내용은 `reference/`로 분리 |
| `reference/sdk_runtime.md` | SDK 호출, robot status, scene_state 제한, pick/place |
| `reference/perception.md` | camera frame, HSV/blob detection, depth 사용 가능성 |
| `reference/motion_navigation.md` | `set_velocity`, `set_head`, vision-only navigation |
| `reference/llm_memory_verification.md` | VLM, LLM JSON decision, memory, verification |
| `reference/implementation_recipes.md` | starter 함수 매핑, 구현 레시피, 피해야 할 패턴 |
| `troubleshooting_runtime.md` | VSCode/Jupyter 실행 중 발견한 runtime 문제와 대응 기록 |

## 핵심 원칙

```text
camera observation
+ color blob detection
+ head scan
+ short velocity control
+ VLM sign reading
+ LLM JSON decision
+ memory-based recovery
```

금지: `scene_state`, 좌표 기반 `go_to`, 정확한 cube/pad entity id 기반 shortcut.
