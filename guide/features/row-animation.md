# 행 애니메이션 (animateRows)

정렬·필터·행 이동으로 표시 행의 위치가 바뀔 때, 이전 위치에서 새
위치로 부드럽게 미끄러지는 전이 애니메이션 (타사 그리드의
`animateRows` 대응).

## 설정

```ts
const grid = new GridCore<User>({ columns, data, animateRows: true });
```

## 코어 API

| API                            | 설명                                               |
| ------------------------------ | -------------------------------------------------- |
| `animateRows?: boolean`        | GridOptions/prop — 행 이동 트랜지션 활성           |
| `grid.isRowAnimationEnabled()` | 활성 여부 — 어댑터가 렌더 후 FLIP 적용 전에 읽는다 |
| `grid.setAnimateRows(on)`      | 런타임 교체 — 어댑터 prop 동기화용 (동일 값 무시)  |

## 동작 방식

코어는 활성 플래그만 소유하고, 실제 움직임은 렌더러가 공용 FLIP
헬퍼(`kernel/rowFlip.ts`의 `captureRowTops`/`applyRowFlip`)로 수행한다:

1. 렌더러가 매 렌더 후 `tr[data-mg-key]`(행 ID)의 `offsetTop`을 기록한다.
2. 다음 렌더에서 이전 위치와 비교해, 양쪽에 모두 있던 행이 이동했으면
   역방향 `translateY`를 심고 다음 프레임에 제거해 CSS 트랜지션
   (`.mg-row-anim`, 180ms)으로 재생한다.

## 동작 규칙 / 주의사항

- 이전 렌더에 없던 행(신규 행, 가상 스크롤 진입)은 애니메이션하지
  않는다 — 스크롤 재렌더에는 반응하지 않는다.
- leaf(데이터) 행과 fullWidth 행이 대상 — 그룹/소계/스켈레톤 행은
  제외된다.
- `prefers-reduced-motion: reduce` 시스템 설정이면 자동 비활성된다.
- 활성 시 tbody에 `mg-animate-rows` 마커 클래스가 붙는다.
- 슬림 엔트리에서는 `animateRowsFeature`를 등록해야 동작한다
  (`features` 옵션).
