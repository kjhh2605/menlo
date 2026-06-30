# LLM, VLM, Memory, Verification Reference

LLM/VLM을 Level 2 agent loop에 안전하게 붙이는 방법과 memory/검증 전략을 정리한다.

## 1. VLM 사용 패턴

워크숍 4의 `ask_vlm`은 JPEG bytes를 base64 data URL로 만들어 LLM에 image+text message를 보낸다.

```python
b64_image = base64.b64encode(jpeg_bytes).decode("utf-8")
image_url = f"data:image/jpeg;base64,{b64_image}"
messages = [{
    "role": "user",
    "content": [
        {"type": "image_url", "image_url": {"url": image_url}},
        {"type": "text", "text": prompt},
    ],
}]
```

Starter에는 `menlo_runner.llm.ask_vlm`과 `ask_vlm_about_frame(ctx, prompt, api_key=...)` wrapper가 있다.

VLM 적합 용도:

- floating warehouse sign 읽기
- 목적지 표지판의 글자/색상/좌우 위치 확인
- pad 근처인지 확인
- color blob detector가 cube와 pad를 혼동할 때 보조 판단

VLM은 느릴 수 있으므로 매 cycle마다 호출하지 말고 다음 상황에 제한한다.

- pad 탐색 stage
- sign이 화면에 들어왔을 가능성이 높을 때
- place 직전 검증
- 반복 실패 후 recovery 판단

## 2. LLM 호출과 메시지 구조

워크숍 4는 Chat Completions 형태의 LLM 호출을 사용한다.

```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": task},
]
reply = call_llm(messages)
```

Starter의 `decide_next_action`에서는 tool-calling 전체 구조보다 단순한 JSON decision이 더 적합하다.

권장 system prompt:

```text
Return ONLY JSON.
Allowed next_action values:
search_cube, navigate_to_cube, pick_cube, search_pad, navigate_to_pad,
place_cube, recover, skip_target, stop.
Schema:
{"next_action": "...", "target_color": null, "reason": "...", "recovery_strategy": null}
```

권장 user payload:

```python
json.dumps(decision_context, ensure_ascii=False)
```

반환값은 반드시 `parse_agent_decision`으로 검증한다.

## 3. Tool-calling agent 구조의 축소 적용

워크숍 4의 agent는 ReAct 구조다.

```text
THINK: LLM이 다음 tool call JSON 생성
ACT: 코드가 tool 실행
OBSERVE: tool result를 다시 message history에 추가
반복
```

워크숍 구성 요소:

- `TOOLS`: 도구 이름과 설명 registry
- `build_system_prompt(tools)`: 사용 가능한 tool과 JSON 호출 형식 설명
- `parse_tool_call(reply)`: fenced code block 안 JSON 추출
- `safe_parse_tool_call(reply)`: parse 실패를 error tool로 변환
- `execute_tool(name, args)`: tool name을 실제 SDK 호출로 매핑
- `run_agent(task, max_turns)`: messages, tool_log 유지 반복

Starter 축소 적용:

| 워크숍 구조 | Starter 구조 |
|---|---|
| tool registry | `ALLOWED_NEXT_ACTIONS` |
| tool call parser | `parse_agent_decision` |
| `execute_tool` | `execute_decision` |
| `tool_log` | `memory.logs` |

## 4. JSON parsing 방어 패턴

워크숍 4는 LLM이 자연어를 섞어 반환할 때 fenced JSON을 추출한다.

```python
match = re.search(r"```(?:json)?\s*(\{.*?\})\s*```", reply, re.DOTALL)
```

Starter의 `parse_agent_decision`도 다음을 처리한다.

- 앞뒤 공백 제거
- fenced code block 제거
- 문자열 중 첫 `{`와 마지막 `}` 추출
- `json.loads`
- `next_action` 허용 목록 검증
- `target_color` 타입 검증

LLM decision 함수는 parser 실패를 가정해야 한다.

권장 fallback:

```python
if parsed is None:
    return AgentDecision(
        next_action="recover",
        reason="LLM returned invalid JSON.",
        recovery_strategy="rescan_and_retry",
    )
```

## 5. Memory와 로그

워크숍 4는 `messages`와 `tool_log`를 유지하며 agent 진행을 디버깅한다.

```python
tool_log.append({
    "turn": turn,
    "tool": tool_name,
    "result": result_text[:100],
})
```

Starter에서는 `AgentMemory.logs`가 같은 역할을 한다.

최소 기록 권장:

- cycle number
- stage
- active_color
- held_color
- visible detections 요약
- LLM decision
- action_result
- verified result
- failed_attempts snapshot

Memory는 단순 로그가 아니라 policy 상태를 바꾸는 기준이어야 한다.

주요 상태 전환:

```text
need_cube -> search_cube -> approaching_cube -> pick_cube
pick success -> holding_cube -> search_pad -> approaching_pad -> place_cube
place success -> need_cube 또는 done
failure -> recover -> 이전 stage 재시도 또는 skip_target
```

## 6. 검증 전략

워크숍에서는 `scene_state`로 성공을 증명하지만, Level 2 최종 logic에는 넣으면 안 된다. 대신 다음 조합을 사용한다.

Navigation 검증:

- target color가 화면 중심 근처인지
- target blob area가 threshold 이상인지
- target을 잃지 않았는지

Pick 검증:

- `pick_entity` result status
- `robot_status.robot.status == "holding"` 또는 유사 상태
- 직후 camera에서 target cube가 더 이상 바닥 중앙에 크게 보이지 않는지

Place 검증:

- `place_entity` result status
- `robot_status`가 holding 상태에서 벗어났는지
- VLM 또는 camera evidence로 matching pad 근처였는지 확인

Recovery 판단:

- target loss
- 같은 action 반복 실패
- pick/place status failure
- navigation max steps 초과
- robot status abnormal/fallen
