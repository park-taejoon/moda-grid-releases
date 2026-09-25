# 가상 스크롤 (Virtual Scroll)

대용량 데이터를 뷰포트만 렌더링한다. DOM에 의존하지 않는 순수 계산기와
GridCore 통합 API로 구성된다.

## DataGrid에서 사용 (가장 간단)

```tsx
<DataGrid columns={columns} data={rows} height={480} rowHeight={37} />
```

`height` + `rowHeight` props를 함께 주면 4개 어댑터 모두 가상화된다 —
스크롤 컨테이너 + 상/하 스페이서 `<tr>`로 `snapshot.virtualRows`만 렌더링하고
헤더는 sticky로 고정된다. `overscan`(기본값 5)으로 여유 행 수 조절.

그룹화·트리 모드에서는 평탄화된 `displayRows` 기준으로 작동한다.

## 코어 API

```ts
const grid = new GridCore({
  columns, data,
  virtualScroll: { rowHeight: 40, viewportHeight: 600 },
});

grid.setVirtualScroll(configOrNull);   // 런타임 활성/비활성 + 설정 변경
grid.setViewportHeight(800);           // 컨테이너 리사이즈 대응
grid.handleScroll(container.scrollTop); // 스크롤 이벤트 → 코어
grid.getVirtualState();                // 현재 VirtualScrollState (비활성 시 null)
```

스냅샷 추가 필드: `virtual`, `virtualRows`
(= `visibleData.slice(virtual.startIndex, virtual.endIndex)`).

- `handleScroll`은 인덱스 범위가 바뀔 때만 알림 — 픽셀 단위 스크롤마다
  리렌더되지 않는다.
- 필터/정렬로 행 수가 바뀌면 인덱스·`totalHeight`가 자동 재계산된다.
- `navigateCell`/`setActiveCell` 후 활성 셀이 뷰포트 밖이면 코어가
  `virtual.scrollTop`을 보정한다 — 어댑터는 이를 DOM `scrollTop`과
  동기화한다.

## 순수 계산기 — `computeVirtualScroll`

커스텀 렌더러나 비-그리드 목록에서도 재사용 가능:

```ts
import { computeVirtualScroll } from "@moda-grid/core";

const v = computeVirtualScroll({
  totalCount: 100_000,
  rowHeight: 40,
  viewportHeight: 600,
  scrollTop: 52_000,
  overscan: 5,           // 기본값 5
});
// v = { startIndex, endIndex, startOffset, totalHeight }
// 렌더링: rows.slice(v.startIndex, v.endIndex)
// 상단 스페이서: v.startOffset px, 컨테이너: v.totalHeight px
```

- `scrollTop`은 `[0, totalHeight - viewportHeight]`로 클램프.
- `totalCount === 0` 또는 `rowHeight <= 0`이면 빈 범위 반환.

## 렌더링 패턴 (커스텀/바닐라 JS)

```html
<div style="height:600px; overflow:auto"
     onscroll="grid.handleScroll(this.scrollTop)">
  <div style="height: {virtual.totalHeight}px; position: relative;">
    <div style="transform: translateY({virtual.startOffset}px)">
      <!-- virtualRows를 rowHeight 고정 높이로 렌더링 -->
    </div>
  </div>
</div>
```
