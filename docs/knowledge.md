# Project Knowledge Index

이 문서는 `workbook`의 4개 워크숍 노트북에서 Level 2 프로젝트 구현에 필요한 기술 배경만 연결한다. 설치, API 키, viewer URL 생성, 세션 생성 같은 세팅 내용은 제외한다.

분석 대상:

- `workbook/workshop_01_basics_ko copy.ipynb`
- `workbook/workshop_02_perception_ko copy.ipynb`
- `workbook/workshop_03_navigation_ko copy.ipynb`
- `workbook/workshop_04_agents_ko.ipynb`

## 빠른 요약

| 주제 | 핵심 | 세부 문서 |
|---|---|---|
| SDK/runtime | async 호출, `ctx` wrapper, robot status, pick/place 제한 | `reference/sdk_runtime.md` |
| Perception | POV JPEG, OpenCV/HSV, blob 중심/면적/각도, depth 선택 사용 | `reference/perception.md` |
| Motion/navigation | `set_velocity`, `set_head`, head scan, vision-only navigation | `reference/motion_navigation.md` |
| LLM/VLM/memory | VLM sign reading, JSON decision, parser 방어, memory/log, 검증 | `reference/llm_memory_verification.md` |
| 구현 레시피 | starter 함수 매핑, cube/pad 처리 순서, 금지 패턴 | `reference/implementation_recipes.md` |

## 최종 구현 원칙

워크숍에는 `scene_state`, 정확한 entity id, 좌표 기반 `go_to` 예제가 포함되어 있다. Level 2 최종 agent logic에서는 이들을 사용하지 않는다. 실제 구현은 다음 조합으로 완성한다.

```text
camera observation
+ color blob detection
+ set_head scan
+ short set_velocity control
+ VLM sign reading
+ LLM JSON decision
+ AgentMemory recovery
```
