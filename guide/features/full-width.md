# 전체 너비 행 (Full-Width Rows)

행 전체가 모든 컬럼을 걸치는 단일 셀로 렌더링되는 기능 — 배너, 구분선,
요약 카드, 공지 행 같은 행 단위 콘텐츠를 데이터 행 사이에 둘 수 있다
(타사 그리드의 `isFullWidthRow` 대응).

## 설정

```ts
const grid = new GridCore<User>({
  columns,
  data,
  // true를 돌려주는 행이 전체 너비 셀로 렌더링된다
  isFullWidthRow: (row, rowIndex) => row.kind === "banner",
});
```

## 코어 API

| API                                           | 설명                                                                                      |
| --------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `isFullWidthRow?: (row, rowIndex) => boolean` | GridOptions — 전체 너비 행 판정 함수. `rowIndex`는 표시 행 기준(필터/정렬/페이징 적용 후) |
| `grid.isFullWidthRow(row, rowIndex)`          | 판정 평가 — 어댑터가 leaf 행마다 호출한다                                                 |
| `grid.setIsFullWidthRow(fn \| null)`          | 런타임 교체 — 어댑터 prop 동기화용 (동일 참조 무시)                                       |

## 어댑터 렌더링

전체 너비 행은 `tr.mg-fullwidth-row > td.mg-fullwidth-cell`(보조
컬럼 포함 전체 colspan)로 렌더된다. 내용은 플랫폼별 렌더 계약으로
채우고, 미지정 시 빈 셀(구분선·스페이서 용도)이 된다.

```tsx
// React — renderFullRow prop
<DataGrid
  isFullWidthRow={(row) => row.id === 2}
  renderFullRow={({ row }) => <div>📢 {row.name}님의 공지</div>}
/>
```

```vue
<!-- Vue 3 / Vue 2 — #fullwidth 스코프드 슬롯 -->
<DataGrid :is-full-width-row="(row) => row.id === 2">
  <template #fullwidth="{ row, rowIndex }">
    <div>📢 {{ row.name }}님의 공지</div>
  </template>
</DataGrid>
```

```svelte
<!-- Svelte 5 — fullwidth 스니펫 -->
<DataGrid {isFullWidthRow}>
  {#snippet fullwidth({ row })}
    <div>📢 {row.name}님의 공지</div>
  {/snippet}
</DataGrid>
```

```ts
// vanilla mountGrid — fullWidthRenderer 옵션 (Node | string 반환)
mountGrid(el, {
  columns,
  data,
  isFullWidthRow: (row) => row.id === 2,
  fullWidthRenderer: ({ row }) => `📢 ${row.name}님의 공지`,
});
```

## 동작 규칙 / 주의사항

- 그룹/소계/스켈레톤 행에는 적용되지 않는다 — 리프(데이터) 행만 판정된다.
- 핀 컬럼과 무관하게 행 전체를 하나의 셀로 렌더한다 — 핀 구역을 따로
  나누지 않는다.
- 셀 선택·편집·셀 병합 대상에서 빠진다 — 행 단위 콘텐츠가 목적이다.
- 가상 스크롤과 병용 가능 — 행 인덱스는 표시 행 기준으로 전달된다.
- `features` 옵션으로 `fullWidthFeature`를 등록하지 않으면
  `grid.isFullWidthRow`는 항상 false(슬림 엔트리 기본).
