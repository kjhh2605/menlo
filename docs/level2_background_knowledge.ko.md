# Level 2 Agent 코드작성 배경지식

이 문서는 다음 코드 작성 에이전트에게 전달할 기술 컨텍스트다. Level 2 프로젝트 조건, 금지/허용 규칙, 제출 제약은 [level2_project_conditions.ko.md](level2_project_conditions.ko.md)에 분리되어 있다.

## 1. Notebook 코드 표면

최종 구현 코드는 `notebooks/project/ko/level_2_autonomous_vision_starter_ko.ipynb`의 starter code cell에 있다.

주요 cell:

- Cell 4: 전체 runnable starter code. 코드 수정이 필요한 주 표면.
- Cell 6: robot context 연결.
- Cell 8: `run_agent(...)` 실행.

Cell 4 안의 주요 함수 위치:

- `AgentMemory`: line 73
- `parse_agent_decision`: line 124
- `decide_next_action`: line 319
- `observe_world`: line 424
- `verify_outcome`: line 450
- `update_memory`: line 539
- `visual_search`: line 627
- `visual_navigate_to_target`: line 647
- `recover_motion`: line 697
- `execute_decision`: line 717
- `run_agent`: line 759

## 2. 현재 코드 구조

Notebook code cell은 크게 네 영역으로 나뉜다.

1. 과제 상수와 LLM decision schema
   - `TASK`
   - `DESTINATION_SIGN_RULES`
   - `SIGNAGE_NOTE`
   - `ALLOWED_NEXT_ACTIONS`
   - `AgentDecision`

2. Memory/observation data model
   - `AgentMemory`
   - `Observation`
   - `ScannedDetection`

3. SDK wrapper와 perception helper
   - `get_robot_status`
   - `get_camera_frame`
   - `ask_vlm_about_frame`
   - `perceive`
   - `set_head`
   - `move_velocity`
   - `pick_nearest_cube`
   - `place_nearest_zone`
   - `scan_head`

4. Agent TODO 구현
   - `decide_next_action`
   - `observe_world`
   - `verify_outcome`
   - `update_memory`
   - `visual_search`
   - `visual_navigate_to_target`
   - `recover_motion`
   - `execute_decision`
   - `run_agent`

## 3. Data model context

`AgentDecision`

- `next_action`: high-level action name.
- `target_color`: action target color or `None`.
- `reason`: decision explanation used for logs/debugging.
- `recovery_strategy`: optional hint for recovery.

`AgentMemory`

- `delivered_count`: notebook-local delivery estimate.
- `delivery_limit`: optional cube cap from completion config.
- `priority_colors`: optional target priority list.
- `held_color`: color believed to be held.
- `active_color`: color currently being searched/navigated.
- `stage`: state machine key.
- `search_turns`: scan/search loop counter.
- `failed_attempts`: failure counter keyed by action/color.
- `completed_colors`: local completed color history.
- `skipped_colors`: skipped target history.
- `logs`: cycle-level evidence for debugging.

`Observation`

- `robot_status`: robot status object from runtime.
- `detections`: list of `ScannedDetection`.
- `note`: observation mode hint, for example `wide_scan` or `front_frame`.
- `vlm_summary`: optional VLM result summary.

`ScannedDetection`

- Wraps color blob fields with the head pose used when the frame was captured.
- `full_bearing_deg` combines image angle and head yaw. Treat it as rough body-relative bearing, not a coordinate.

## 4. State machine context

Current code uses `memory.stage` as the main execution state.

Observed stages:

- `need_cube`
- `search_cube`
- `approaching_cube`
- `ready_to_pick`
- `holding_cube`
- `search_pad`
- `approaching_pad`
- `ready_to_place`
- `recover`
- `done`

Expected state flow:

```text
need_cube/search_cube
  -> navigate_to_cube
  -> ready_to_pick
  -> pick_cube
  -> holding_cube
  -> search_pad/navigate_to_pad
  -> ready_to_place
  -> place_cube
  -> need_cube
```

Failure flow:

```text
action failure
  -> failed_attempts[action:color] += 1
  -> recover when repeated
  -> if holding cube: search_pad
  -> otherwise: search_cube
```

## 5. Decision context passed to LLM

`build_decision_context` produces the JSON-like context used by `decide_next_action`.

Fields currently included:

- `task`
- `visible_targets`
- `held_color`
- `active_color`
- `stage`
- `delivered_count`
- `completed_colors`
- `skipped_colors`
- `failed_attempts`
- `last_result`
- `note`
- `signage_note`
- `vlm_summary`

`visible_targets` contains compact detection summaries:

- `color`
- `angle_deg`
- `full_bearing_deg`
- `blob_area`
- `bbox`

This is enough for high-level target/action selection. It should not be expanded with raw image bytes or low-level control commands.

## 6. LLM decision implementation context

`decide_next_action` currently has two paths.

LLM path:

- Builds a system prompt.
- Sends `decision_context` as JSON.
- Requires one JSON object as output.
- Parses output with `parse_agent_decision`.
- Applies `stage_safe` filtering.

Fallback path:

- Used when API key is missing, LLM call fails, JSON is invalid, or stage safety rejects the output.
- Uses `memory.stage`, `held_color`, visible detections, and failure count to select a conservative action.

Important implementation detail:

- `stage_safe` mutates `parsed.target_color` to `memory.held_color` when holding a cube and the model omits target color.
- If holding a cube, cube-search/pick actions are rejected.
- If not holding a cube, pad-search/place actions are rejected.
- Manipulation actions are rejected unless the stage is already ready.

## 7. Perception context

`perceive(ctx)` reads the POV camera and calls `detect_color_blobs(jpeg)`.

The detector returns color blobs with:

- `color`
- `angle_deg`
- `blob_area`
- `centroid`
- `bbox`

`scan_head(ctx, yaws=..., pitch=...)`:

- Moves head to each yaw.
- Waits briefly.
- Runs `perceive`.
- Wraps each detection as `ScannedDetection`.

Interpretation hints:

- `blob_area` is only a proximity/visibility proxy.
- `angle_deg` is useful for short yaw correction.
- Same color can refer to cube, pad background, signage, or other colored surfaces depending on stage.
- `held_color` and `stage` are necessary to interpret what a color blob probably means.

## 8. Visual search context

`visual_search(ctx, target_color=None)`:

- Runs several yaw scan patterns.
- Filters detections by `target_color` when provided.
- Chooses the largest candidate.
- Sets head toward the rough bearing.
- If no candidate is found, performs a short body turn/forward nudge and retries.

Useful tuning variables:

- scan yaw tuples
- pitch
- body turn direction/magnitude
- nudge duration
- candidate ranking function

Potential weaknesses:

- Largest blob can be a pad/background rather than cube.
- Wide scan can lose target after body motion.
- Search does not currently classify source area vs destination area.

## 9. Visual navigation context

`visual_navigate_to_target(ctx, target_color)`:

- Centers head.
- Repeatedly observes the front frame.
- Selects the largest blob matching `target_color`.
- If missing, rotates briefly to reacquire.
- If angle is outside tolerance, applies short yaw correction.
- If centered, moves forward.
- Returns success when area is large enough and target is roughly centered.

Important local variables:

- `arrival_area`
- `centered_tolerance_deg`
- `last_seen_area`
- loop count
- forward speed
- yaw correction speed
- command duration

The function uses area-based arrival. That means arrival threshold is the main lever for pick/place readiness.

## 10. Verification context

`verify_outcome` combines:

- SDK action result summary
- post-action scan evidence
- current memory estimates

It returns:

- `ok`
- `decision`
- `action_result`
- `delivered_count`
- `held_cube`
- `held_color`
- `sdk_status`
- `sdk_error`
- `post_visible_count`
- `target_seen`
- `max_target_area`

Implementation nuance:

- Search/navigation success comes from action result booleans.
- Pick/place success is inferred from SDK result status/error and precondition block status.
- `held_color` is set after successful pick.
- `delivered_count` is incremented after successful place.

## 11. Memory update context

`update_memory` is the central state transition point.

Failure key format:

```text
{next_action}:{color or "none"}
```

Success behavior:

- Clears the matching failure key.
- Advances stage according to action.
- Updates held/active color.
- Appends completed color after place.
- Appends structured log entry.

Failure behavior:

- Increments failure key.
- Moves stage back to search state where appropriate.

Recovery behavior:

- If holding cube, return to `search_pad`.
- Otherwise return to `search_cube`.

## 12. Recovery context

`recover_motion` currently:

- Cancels current action.
- Steps backward.
- Applies a short turn.
- Scans with a wider head pattern.
- Returns whether target color is visible.

Inputs:

- `memory.failed_attempts`
- `memory.held_color`
- `memory.active_color`
- optional `reason`

Potential improvements:

- Clear only the worst repeated failure, not all failures.
- Alternate wider body turns after repeated recoveries.
- If held cube remains true, avoid cube-search behavior.
- Use `skipped_colors` when a target repeatedly cannot be recovered.

## 13. Runtime context

The notebook depends on the runtime support package installed by the setup cell.

Runtime objects expected by the code:

- `ctx.get_vision("pov")`
- `ctx.state("robot_status")`
- `ctx.invoke("set_head", ...)`
- `ctx.invoke("set_velocity", ...)`
- `ctx.invoke("pick_entity", ...)`
- `ctx.invoke("place_entity", ...)`
- `ctx.invoke("cancel", {})`

Environment variables:

- `MENLO_API_KEY`: robot/session connection.
- `TOKAMAK_API_KEY`: text LLM and optional VLM.

If the LLM call fails, fallback should keep the agent loop alive.

## 14. Debugging context

Useful evidence in logs:

- `stage`
- `active_color`
- `held_color`
- `delivered_count`
- visible color set
- decision dict
- verification dict
- failure counter snapshot

When runtime behavior is bad, inspect logs by failure type:

- repeated `search_cube:*`: target not being found or scan too narrow.
- repeated `navigate_to_cube:*`: area threshold/centering/approach issue.
- repeated `pick_cube:*`: readiness too early or SDK manipulation failure.
- repeated `search_pad:*`: pad color/sign not being found.
- repeated `navigate_to_pad:*`: same-color confusion or threshold issue.
- repeated `place_cube:*`: ready_to_place too early or wrong pad.

## 15. Cross-reference

Use this document for code structure and implementation context.

Use [level2_project_conditions.ko.md](level2_project_conditions.ko.md) for:

- Level 2 project constraints
- hard bans
- allowed control surface
- notebook-only working rule
- validation obligations
- residual runtime risk language

