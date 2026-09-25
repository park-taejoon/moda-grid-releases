# 페이징 (Paging)

로컬 데이터를 페이지 단위로 나눠 표시한다. **페이징 UI는 DataGrid에 없다** —
앱에서 스냅샷으로 직접 구현한다.

## 코어 API

```ts
new GridCore({ columns, data, pageSize: 20 });   // 초기 페이지 크기
// 또는 런타임:
grid.setPage(0, 20);        // pageIndex 0, 크기 20 (size=0이면 비활성)
grid.setPage(2);            // 3페이지로 이동
grid.setPage(snapshot.pageIndex + 1);  // 다음 페이지 — 스냅샷 기준 계산
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
  >‹</button>
  <span>{snapshot.pageIndex + 1} / {snapshot.pageCount}</span>
  <button
    disabled={snapshot.pageIndex + 1 >= snapshot.pageCount}
    onClick={() => grid.setPage(snapshot.pageIndex + 1)}
  >›</button>
  <select
    value={snapshot.pageSize}
    onChange={(e) => grid.setPage(0, Number(e.target.value))}
  >
    {[10, 20, 50, 100].map((n) => <option key={n} value={n}>{n}행</option>)}
  </select>
</div>
```

Vue/Svelte도 동일 — `state.value.pageIndex`(Vue) / `$store.pageIndex`
(Svelte)로 읽고 `grid.setPage`로 변경한다.
