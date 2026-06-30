# Perception Reference

Camera frame에서 색상, 방향, 거리 단서를 추출하는 방법을 정리한다.

## 1. Camera frame 처리

`get_vision("pov")`는 JPEG bytes를 반환한다.

```python
jpeg = await session.get_vision("pov")
img = cv2.imdecode(np.frombuffer(jpeg, np.uint8), cv2.IMREAD_COLOR)
```

OpenCV는 BGR 순서를 사용한다. 표시하거나 PIL로 넘길 때는 RGB로 바꾼다.

```python
pil_img = PIL.Image.fromarray(cv2.cvtColor(img, cv2.COLOR_BGR2RGB))
```

중앙 관심 영역만 잘라 확인할 수 있다.

```python
h, w = img.shape[:2]
roi = img[h//3 : 2*h//3, w//3 : 2*w//3]
```

프로젝트에서는 화면 전체 detection을 기본으로 쓰고, 접근 완료/정렬 판단에는 중심 영역 또는 centroid 위치를 추가로 쓰면 좋다.

## 2. HSV 색상 검출

워크숍 2/3의 핵심 perception 방식은 HSV thresholding이다.

```python
hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)

COLOR_RANGES = {
    "red": [
        (np.array([0, 50, 50]), np.array([10, 255, 255])),
        (np.array([160, 50, 50]), np.array([180, 255, 255])),
    ],
    "green": [
        (np.array([40, 50, 50]), np.array([80, 255, 255])),
    ],
    "blue": [
        (np.array([100, 50, 50]), np.array([130, 255, 255])),
    ],
    "yellow": [
        (np.array([20, 50, 50]), np.array([35, 255, 255])),
    ],
}
```

Red는 hue가 0/180 경계에 걸치므로 두 범위를 OR로 합친다.

마스크 생성:

```python
mask = np.zeros(hsv.shape[:2], dtype=np.uint8)
for lo, hi in ranges:
    mask |= cv2.inRange(hsv, lo, hi)
```

윤곽선 찾기:

```python
contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
significant = [c for c in contours if cv2.contourArea(c) > 200]
largest = max(significant, key=cv2.contourArea)
```

`200px²` 정도는 작은 noise를 버리는 기본 threshold다. 최종 프로젝트에서는 카메라 거리와 조명에 따라 조정 가능하다.

## 3. Blob 중심점, 면적, 각도

가장 큰 contour 중심점은 image moment로 계산한다.

```python
M = cv2.moments(largest)
if M["m00"] != 0:
    cx = int(M["m10"] / M["m00"])
    cy = int(M["m01"] / M["m00"])
```

카메라 중심 기준 수평 각도:

```python
HFOV_HALF_DEG = 30.0
angle_deg = (cx - w / 2) / (w / 2) * HFOV_HALF_DEG
```

의미:

- `angle_deg > 0`: target이 화면 오른쪽. 몸을 오른쪽으로 돌리려면 일반적으로 `wz < 0`.
- `angle_deg < 0`: target이 화면 왼쪽. 몸을 왼쪽으로 돌리려면 일반적으로 `wz > 0`.
- `abs(angle_deg) <= 5~10`: 정렬 완료 후보.

`blob_area`는 거리의 대리값이다.

- area가 작다: 멀다.
- area가 크다: 가깝다.
- `arrival_area`를 넘으면 접근 완료로 판단한다.

워크숍 3 예시는 `ARRIVAL_AREA = 8000`을 사용한다. 실제 프로젝트에서는 큐브/패드 크기, 카메라 pitch, target 종류에 따라 튜닝한다.

## 4. Perceive 함수 구조

워크숍 2/3의 `perceive(session)` 구조:

```python
async def perceive(session):
    jpeg = await session.get_vision("pov")
    img = cv2.imdecode(np.frombuffer(jpeg, np.uint8), cv2.IMREAD_COLOR)
    h, w = img.shape[:2]
    hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)

    result = {}
    for color, ranges in COLOR_RANGES.items():
        mask = ...
        contours = ...
        largest = ...
        area = cv2.contourArea(largest)
        cx = ...
        angle_deg = (cx - w / 2) / (w / 2) * 30.0
        result[color] = {
            "angle_deg": round(angle_deg, 1),
            "blob_area": int(area),
        }
    return result
```

Starter에서는 `menlo_runner.perception.detect_color_blobs(jpeg)`가 비슷한 역할을 한다. 반환 detection에는 색상, 각도, 면적, centroid, bbox 등이 포함된다.

따라서 starter에서는 직접 OpenCV를 다시 구현하기보다 우선 `detect_color_blobs`와 `ScannedDetection`을 활용하고, 필요할 때 bbox/centroid 기반 판단을 추가한다.

## 5. Depth Anything 사용 가능성

워크숍 2는 `transformers.pipeline("depth-estimation")`으로 Depth Anything v2를 사용한다.

```python
from transformers import pipeline as hf_pipeline

depth_pipe = hf_pipeline(
    "depth-estimation",
    model="depth-anything/Depth-Anything-V2-Small-hf",
)
depth_result = depth_pipe(pil_img)
depth_map = np.array(depth_result["depth"])
```

색상 blob 중심점에서 depth score를 읽는다.

```python
depth_score = float(depth_map[cy, cx])
```

워크숍 기준으로 Depth Anything에서는 값이 클수록 가까운 것으로 다룬다.

적용 판단:

- 장점: blob 면적보다 거리 순서 판단이 나을 수 있다.
- 단점: 모델 로딩과 추론이 무겁고 느릴 수 있다.
- 권장: 기본 navigation은 `blob_area`와 angle로 구현하고, depth는 가까운 target 선택이 불안정할 때 선택적으로 사용한다.
