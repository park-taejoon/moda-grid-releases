# 페이징 (Paging)

로컬 데이터를 페이지 단위로 나눠 표시한다. **페이징 UI는 DataGrid에 없다** —
앱에서 스냅샷으로 직접 구현한다.

## 코어 API

```ts
new GridCore({ columns, data, pageSize: 20 }); // 초기 페이지 크기
// 또는 런타임:
grid.setPage(0, 20); // pageIndex 0, 크기 20 (size=0이면 비활성)
grid.setPage(2); // 3페이지로 이동
grid.setPage(snapshot.pageIndex + 1); // 다음 페이지 — 스냅샷 기준 계산
```

## 스냅샷 필드

`snapshot.pageIndex` / `snapshot.pageSize` / `snapshot.pageCount` /
`snapshot.filteredRowCount`(페이징 전 행 수).

## 동작 규칙

- 파이프라인의 **마지막 단계** — 필터 → 검색 → 정렬 후 `slice`.
- 필터/검색/정렬 조건 변경 시 `pageIndex`는 0으로 리셋된다.
- 데이터가 줄어 현재 페이지가 범위를 벗어나면 마지막 페이지로 자동 보정.
- `pageSize: 0`이면 페이징 비활성 (전체 출력).
- 서버 사이드 모드·그룹화와는 병용되지 않는다
  (서버 모드는 무한 스크롤, 그룹 모드는 평탄 리스트로 동작).

## 페이지 툴바 예시 (React)

```tsx
const { grid, snapshot } = useGridCore({ columns, data, pageSize: 20 });

<div className="pager">
  <button
    disabled={snapshot.pageIndex === 0}
    onClick={() => grid.setPage(snapshot.pageIndex - 1)}
  >
    ‹
  </button>
  <span>
    {snapshot.pageIndex + 1} / {snapshot.pageCount}
  </span>
  <button
    disabled={snapshot.pageIndex + 1 >= snapshot.pageCount}
    onClick={() => grid.setPage(snapshot.pageIndex + 1)}
  >
    ›
  </button>
  <select
    value={snapshot.pageSize}
    onChange={(e) => grid.setPage(0, Number(e.target.value))}
  >
    {[10, 20, 50, 100].map((n) => (
      <option key={n} value={n}>
        {n}행
      </option>
    ))}
  </select>
</div>;
```

Vue/Svelte도 동일 — `state.value.pageIndex`(Vue) / `$store.pageIndex`
(Svelte)로 읽고 `grid.setPage`로 변경한다.

## `mountGrid` 내장 페이저

`pager: true`면 이전/다음 버튼 + **페이지 번호 버튼**(현재 기준 ±2, 최소
5칸 창 — `.mg-pager-pages`/`.mg-pager-num`, 활성 페이지는 `.mg-active`)을
렌더링한다. `pageSizeOptions`를 주면 페이지 크기 셀렉트(`.mg-pager-size`)도
추가된다:

```ts
mountGrid(el, {
  columns,
  data,
  pager: true,
  pageSize: 20,
  pageSizeOptions: [10, 20, 50, 100],
});
```

## `autoPageSize` — 페이지 크기 자동 계산

뷰포트 높이(`height`)와 행 높이(`rowHeight`)로 한 화면에 들어가는 행 수를
자동 계산해 `pageSize`로 설정한다. 헤더·페이저 높이는 실측으로 차감하고,
컨테이너 리사이즈 시 재계산한다.

```ts
mountGrid(el, {
  columns,
  data,
  height: 480,
  rowHeight: 37,
  pager: true,
  autoPageSize: true, // pageSize = floor((480 - 헤더 - 페이저) / 37)
});
```

어댑터도 동일한 prop (`autoPageSize` / `auto-page-size`)을 제공한다.
수동 `setPage(i, size)` 호출과 공존 가능하지만 리사이즈 시 자동값으로
덮어쓴다.

## `domLayout: "autoHeight"` — 내용만큼 높이

`domLayout: "autoHeight"`이면 내부 스크롤 컨테이너의 높이 고정을 해제해
그리드가 표시 행 수만큼 늘어난다. `height`와 가상 스크롤은 무시된다
(페이지 크기로 행 수를 제한하는 `pager`/`autoPageSize`와 조합 권장).

```ts
mountGrid(el, {
  columns,
  data,
  domLayout: "autoHeight",
  pager: true,
  autoPageSize: false, // pageSize 미지정 시 전체 행이 한 화면에 펼쳐진다
});
```

어댑터 prop: `domLayout="autoHeight"` (React), `dom-layout="autoHeight"`
(Vue), `domLayout="autoHeight"` (Svelte).
