# SDK and Runtime Reference

Level 2 구현에서 SDK를 호출하는 방식과 사용할 수 있는 상태 정보를 정리한다.

## 1. Async 호출 모델

Menlo SDK와 로봇 runtime 호출은 대부분 async 방식이다. Notebook 셀 최상위에서는 바로 `await`를 사용할 수 있지만, 함수 안에서는 `async def`로 감싼다.

기본 호출 형태:

```python
result = await session.invoke("skill_name", args, timeout_s=30)
state = await session.state.get("robot_status")
jpeg = await session.get_vision("pov")
```

Starter는 `session` 대신 `ctx` wrapper를 사용한다.

```python
await ctx.invoke("set_velocity", {"vx": vx, "vy": vy, "wz": wz, "duration_s": duration_s})
await ctx.state("robot_status")
await ctx.get_vision("pov")
```

구현 시 starter wrapper를 우선 사용한다.

- `get_robot_status`
- `get_camera_frame`
- `set_head`
- `move_velocity`
- `pick_nearest_cube`
- `place_nearest_zone`

## 2. Robot status

`robot_status`는 로봇 자체와 runtime 상태를 확인하는 기본 정보다.

자주 쓰는 필드:

| 필드 | 용도 |
|---|---|
| `state.robot.status` | 예: `ready`, `holding`, `fallen` |
| `state.robot.entity_id` | 현재 로봇 entity id |
| `state.robot.pose.position` | `[x, y, z]` 형태 위치 |
| `state.robot.pose.yaw_deg` | world frame 기준 yaw |
| `state.runtime.status` | runtime 준비 상태 |
| `state.runtime.accepts_tool_calls` | command 수신 가능 여부 |
| `state.robot.extra["head"]` | `set_head` 이후 target/measured yaw, pitch 확인 |

Level 2 최종 구현에서 `robot_status`는 허용되지만, 절대 좌표 하드코딩이나 좌표 기반 `go_to`처럼 쓰면 안 된다. 상태 확인, 실패 검증, 로그 기록에 제한해서 사용한다.

## 3. `scene_state`의 의미와 제한

워크숍 1/3/4는 `scene_state`를 많이 사용한다. 여기에는 전체 entity 정보가 들어 있다.

관찰 가능한 구조:

- `scene.entities`: entity id -> entity 객체
- `entity.pose.position`: entity 위치
- `entity.pose.yaw_deg`: entity 방향
- `entity.visible`: 보이는 큐브 여부
- `entity.state["color"]`: 큐브 색상
- `entity.state["parent_pad_id"]`: 큐브가 놓인 pad
- `entity.attached_to`: 로봇이 들고 있는 큐브 확인

하지만 Level 2 최종 agent logic에서는 `scene_state`, 정확한 cube/pad entity id, 좌표 기반 `go_to`를 사용하면 안 된다.

허용 용도:

- 개발 중 개념 이해
- 디버깅 또는 오프라인 검증
- 발표에서 baseline과 Level 2 방식 차이 설명

최종 agent logic은 camera, head scan, velocity command, memory, LLM/VLM 중심으로 구현한다.

## 4. Pick and place

워크숍의 pick 예시:

```python
await session.invoke(
    "pick_entity",
    {"target": {"kind": "entity", "entity_id": "cube"}},
    timeout_s=300,
)
```

`entity_id="cube"`는 정확한 cube id가 아니라 가까운 큐브 alias로 쓰인다. 호출 전에는 반드시 camera 기반 navigation으로 의도한 큐브 근처까지 이동한다.

워크숍의 place 예시:

```python
await session.invoke(
    "place_entity",
    {"target": {"kind": "entity", "entity_id": target_pad}},
    timeout_s=300,
)
```

Starter는 Level 2 제약에 맞게 가까운 zone에 내려놓는 wrapper를 제공한다.

```python
await ctx.invoke("place_entity", {}, timeout_s=300)
```

프로젝트에서는 matching pad 근처까지 vision-only로 이동한 뒤 `place_nearest_zone(ctx)`를 호출한다.

검증 조합:

- `result.status`
- `result.error`
- 이후 camera 관찰
- `robot_status`
