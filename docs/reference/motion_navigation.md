# Motion and Navigation Reference

Level 2에서 허용되는 `set_velocity`, `set_head`, camera feedback 기반 navigation 패턴을 정리한다.

## 1. Velocity command

`set_velocity`는 짧은 시간 동안 body-frame velocity를 보내는 이동 primitive다.

```python
await session.invoke(
    "set_velocity",
    {"vx": 0.5, "vy": 0.0, "wz": 0.0, "duration_s": 1.0},
    timeout_s=30,
)
```

파라미터:

| 파라미터 | 의미 |
|---|---|
| `vx` | 전진/후진 속도. 양수는 전진 |
| `vy` | 좌우 이동 속도 |
| `wz` | 회전 속도(rad/s). 양수는 왼쪽, 음수는 오른쪽 회전 |
| `duration_s` | 명령 지속 시간 |

워크숍 기준 주의점:

- `|vx|`, `|vy|`는 대략 `1.5` 이하로 제한된다.
- `|wz|`는 대략 `0.6` 이하로 제한된다.
- 제한보다 큰 값은 조용히 clipping될 수 있다.
- 이 로봇은 제자리 회전이 잘 되지 않으므로 회전할 때 작은 `vx`를 같이 넣는 것이 좋다.

회전 시간 계산:

```python
duration = abs(heading_error_deg) / (abs(wz) * 57.296)
```

`57.296`은 1 radian을 degree로 바꾼 값이다. 예: `wz=0.5`로 90도 회전하려면 약 `3.14s`.

Level 2에서는 큰 이동 명령 하나보다 다음 패턴이 안전하다.

```text
observe -> short move -> observe -> correct -> repeat
```

## 2. Head control과 scan

`set_head`는 몸 방향은 그대로 두고 카메라 방향만 조정한다.

```python
await session.invoke("set_head", {"yaw": 0.6, "pitch": 0.15})
```

Starter wrapper:

```python
await set_head(ctx, yaw=0.6, pitch=0.15)
```

활용:

- `yaw=0.0`: 정면 보기
- 양/음 yaw: 좌우 scan
- `pitch=0.15`: 바닥 큐브/패드 관찰을 위해 약간 아래 보기

Starter의 `scan_head` 구조:

```python
for yaw in (-0.8, 0.0, 0.8):
    await set_head(ctx, yaw=yaw, pitch=0.15)
    await asyncio.sleep(0.4)
    detections = await perceive(ctx)
```

`ScannedDetection.full_bearing_deg`는 image angle과 head yaw를 합쳐 body 기준 bearing으로 쓰기 위한 값이다.

## 3. Global navigation 원리와 금지 범위

워크숍 3 Part A는 `go_to`를 전역 좌표로 재구현한다. Level 2 최종 구현에서는 금지지만, 제어 원리 이해에는 유용하다.

목표까지 거리:

```python
dist = math.hypot(tx - rx, ty - ry)
```

목표 bearing:

```python
bearing = math.degrees(math.atan2(ty - ry, tx - rx))
```

Heading error 정규화:

```python
heading_error = (bearing - yaw + 180) % 360 - 180
```

`turn_to_face` 패턴:

```python
wz = 0.4 if heading_error > 0 else -0.4
duration = min(abs(heading_error) / (abs(wz) * 57.296), 8.0)
await session.invoke("set_velocity", {"vx": 0.15, "wz": wz, "duration_s": duration})
```

`drive_to_distance` 패턴:

```python
vx = min(0.8, dist * 0.4)
await session.invoke("set_velocity", {"vx": vx, "wz": 0.0, "duration_s": 0.8})
```

Level 2 적용 방식: 좌표 대신 camera angle과 blob area를 feedback으로 바꿔 같은 구조를 재사용한다.

## 4. Vision-only navigation

워크숍 3 Part B의 핵심은 camera만으로 `go_to`와 비슷한 동작을 만드는 것이다.

### 4.1 Center on color

```python
async def center_on_color(session, target_color, angle_tolerance=10.0, max_steps=12):
    for step in range(max_steps):
        obs = await perceive(session)
        if target_color not in obs:
            await session.invoke(
                "set_velocity",
                {"vx": 0.25, "vy": 0.0, "wz": 0.3, "duration_s": 1.0},
            )
            continue

        angle = obs[target_color]["angle_deg"]
        if abs(angle) <= angle_tolerance:
            return True

        wz = -0.3 if angle > 0 else 0.3
        await session.invoke(
            "set_velocity",
            {"vx": 0.25, "vy": 0.0, "wz": wz, "duration_s": 0.8},
        )
    return False
```

핵심:

- target이 안 보이면 천천히 전진+회전하며 scan한다.
- target이 보이면 image angle 부호에 따라 회전한다.
- target이 중앙 tolerance 안에 들어오면 정렬 성공이다.

### 4.2 Drive toward color

```python
async def drive_toward_color(session, target_color, arrival_area=8000, max_steps=15):
    for step in range(max_steps):
        obs = await perceive(session)
        if target_color not in obs:
            return False

        area = obs[target_color]["blob_area"]
        angle = obs[target_color]["angle_deg"]

        if area >= arrival_area:
            return True

        if step % 2 == 1 and abs(angle) > 10:
            wz = -0.3 if angle > 0 else 0.3
            await session.invoke("set_velocity", {"vx": 0.15, "wz": wz, "duration_s": 0.5})
        else:
            await session.invoke("set_velocity", {"vx": 0.5, "wz": 0.0, "duration_s": 1.0})
    return False
```

핵심:

- 매 step마다 다시 관찰한다.
- `blob_area`가 충분히 커지면 도착으로 판단한다.
- angle이 커지면 전진 대신 짧게 재정렬한다.
- target을 잃으면 실패를 반환하고 recovery로 넘긴다.

## 5. Starter 적용 흐름

`visual_navigate_to_target(ctx, target_color)` 권장 흐름:

```text
1. scan_head로 target_color 찾기
2. full_bearing_deg 또는 angle_deg로 body 방향 정렬
3. 짧게 set_velocity 전진
4. 다시 perceive
5. area/bbox/centroid로 접근 완료 판단
6. target loss면 recover_motion으로 넘김
```
