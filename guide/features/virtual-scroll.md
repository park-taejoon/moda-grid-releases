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
  columns,
  data,
  virtualScroll: { rowHeight: 40, viewportHeight: 600 },
});

grid.setVirtualScroll(configOrNull); // 런타임 활성/비활성 + 설정 변경
grid.setViewportHeight(800); // 컨테이너 리사이즈 대응
grid.handleScroll(container.scrollTop); // 스크롤 이벤트 → 코어
grid.getVirtualState(); // 현재 VirtualScrollState (비활성 시 null)
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
  overscan: 5, // 기본값 5
});
// v = { startIndex, endIndex, startOffset, totalHeight }
// 렌더링: rows.slice(v.startIndex, v.endIndex)
// 상단 스페이서: v.startOffset px, 컨테이너: v.totalHeight px
```

- `scrollTop`은 `[0, totalHeight - viewportHeight]`로 클램프.
- `totalCount === 0` 또는 `rowHeight <= 0`이면 빈 범위 반환.

## 렌더링 패턴 (커스텀/바닐라 JS)

```html
<div
  style="height:600px; overflow:auto"
  onscroll="grid.handleScroll(this.scrollTop)"
>
  <div style="height: {virtual.totalHeight}px; position: relative;">
    <div style="transform: translateY({virtual.startOffset}px)">
      <!-- virtualRows를 rowHeight 고정 높이로 렌더링 -->
    </div>
  </div>
</div>
```

## 컬럼 가상화 — `virtualColumns`

수백 개 컬럼을 전부 DOM에 만들지 않고, 수평 뷰포트(+overscan) 주변
컬럼만 렌더링한다. 행 가상화와 달리 컬럼은 가변 너비라 누적 너비로
윈도우를 계산한다 (`computeVirtualColumns` — `virtualColumns.ts`).

```tsx
<DataGrid columns={columns} data={rows} virtualColumns />
// overscan 조절 (기본값 2개 컬럼)
<DataGrid columns={columns} data={rows} virtualColumns={{ overscan: 4 }} />
```

vanilla `mountGrid`는 `virtualColumns: true` 옵션으로 동일하게 켠다.

### DOM 계약 (5개 렌더러 공통)

- 숨겨진 연속 컬럼 구간은 `td.mg-col-spacer[colSpan]` 하나로 대체된다 —
  colgroup이 전 컬럼을 유지하므로 `table-layout: fixed`에서 너비 합이
  자동 상속된다.
- `pinned: "left" | "right"` 컬럼은 sticky라 윈도우와 무관하게 항상 렌더된다.
- 헤더/필터 행/데이터 행/고정 행/총계 행이 모두 같은 윈도우를 공유한다.
- 멀티레벨 헤더(`columnGroups`)는 colSpan 정렬 문제로 윈도우를 끄고
  전체 컬럼을 렌더한다.

### 코어 API

```ts
grid.setVirtualColumns(true | { overscan: 4 } | null); // 런타임 토글
grid.handleScroll(el.scrollTop, el.scrollLeft); // 스크롤 이벤트 → 코어
grid.setColumnViewport(el.clientWidth, leadWidth); // 마운트/리사이즈 보고
// leadWidth = 데이터 컬럼 앞의 보조 셀(상태/체크/행번호) 너비 합
grid.scrollToColumn(150); // 컬럼이 보이게 스크롤
grid.scrollToCell(20, 150); // 셀 단위도 동일 경로
```

스냅샷 추가 필드: `virtualCols` — `{ startIndex, endIndex, scrollLeft }`
(`visibleColumns` 기준). 어댑터는 이를 `buildColumnWindow`에 넣어
`col`/`spacer` 렌더 플랜을 얻고, `scrollLeft`를 DOM과 동기화한다.

- 키보드 내비(화살표/Tab/Ctrl+End)·`scrollToColumn`은 화면 밖 컬럼을
  자동으로 노출한다 — `scrolling.requestScroll`이
  `ensureColumnVisible`을 선행 호출해 `scrollLeft`를 보정한다.
- 뷰포트 미측정(`viewportWidth <= 0`) 또는 전체 너비가 뷰포트보다
  좁으면 전체 컬럼을 렌더한다 — 부트스트랩·소형 그리드에서 빈 화면 방지.
- `domLayout: "autoHeight"`에서도 가로 스크롤은 유지된다 — 세로만
  내용 높이에 맞춰 늘어난다.

## 대량 변경 묶기 — `grid.batch(fn)`

여러 변경 API를 연속 호출하면 매번 파이프라인(필터→정렬→그룹→페이징)이
재계산되고 스냅샷이 발행된다. `batch`로 묶으면 **블록이 끝날 때 한 번만**
재계산·발행된다 — 대량 행 추가/숨김/정렬 변경 같은 배치 작업에 유용하다
(타사 그리드의 batch 업데이트와 동일 목적):

```ts
grid.batch(() => {
  incoming.forEach((r) => grid.addRows(r));
  grid.setRowHidden(idsToHide, true);
  grid.setSort("name", "asc");
}); // 여기서 딱 한 번만 refresh + notify
```

- 중첩 `batch`는 바깥이 끝날 때 한 번만 갱신된다.
- 블록이 예외로 끝나도 pending 갱신은 flush된다 (finally 보장).
- 블록 안에서 `getSnapshot()`은 **마지막 갱신 전 상태**를 반환할 수
  있다 — batch는 읽기가 아니라 쓰기 묶음으로 사용한다.
