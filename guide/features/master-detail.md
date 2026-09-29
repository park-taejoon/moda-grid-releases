# 행 상세 패널 (Master-Detail)

각 행 아래에 전체 너비의 상세 패널을 펼치는 기능. 첫 컬럼에 `▸/▾` 토글
버튼이 자동으로 생기고, 펼침 상태는 `getRowId` 기준으로 관리된다.

## 설정

렌더러를 지정하면 기능이 활성화된다 — 지정하지 않으면 토글도 없다.

```ts
// vanilla
mountGrid(el, {
  columns,
  data,
  detailRenderer: (row, rowIndex) => Node | string | null,
  detailHeight: 120, // 선택 — 미지정 시 내용에 맞춤
});
```

```tsx
// React — detailRenderer prop이 ReactNode를 반환
<DataGrid
  columns={columns}
  data={data}
  detailRenderer={(row, rowIndex) => <div>{row.name} 상세 내용</div>}
  detailHeight={120}
/>
```

```vue
<!-- Vue 3 / Vue 2 — #detail 스코프 슬롯 -->
<DataGrid :columns="columns" :data="data">
  <template #detail="{ row, rowIndex }">
    <div>{{ row.name }} 상세 내용</div>
  </template>
</DataGrid>
```

```svelte
<!-- Svelte 5 — detail snippet -->
<DataGrid {columns} {data}>
  {#snippet detail({ row, rowIndex })}
    <div>{row.name} 상세 내용</div>
  {/snippet}
</DataGrid>
```

## 코어 API

```ts
grid.toggleRowDetail(rowOrId); // 펼침/접힘 토글
grid.isDetailExpanded(rowOrId); // 펼침 여부
grid.setDetailExpanded(rowOrId, true); // 상태 지정 (같은 값이면 무시)
grid.getDetailExpandedIds(); // 펼친 행 ID 목록
```

`rowOrId`는 `getRowId` 기준 행 ID 문자열 또는 행 객체 모두 받는다.

## 동작 규칙

- 토글은 `detailRenderer`/`#detail`/`detail` 슬롯이 설정된 경우에만 첫
  컬럼 셀에 렌더링된다.
- 상세 행은 마스터 행 바로 아래 `tr.mg-detail-row > td.mg-detail-cell`
  (전체 컬럼 colspan)으로 렌더링되고, 데이터 행 수·선택·편집에는 포함되지
  않는다.
- 펼침 상태는 스냅샷 `detailExpanded: ReadonlySet<string>`에 노출된다.
- 아직 데이터에 없는 ID도 기록할 수 있다 — 서버 사이드/append 모드에서
  나중에 도착한 행에도 적용된다.
- 펼침 상태는 `grid.getState()` / `grid.applyState()`로 지속된다
  (`detailExpanded` 필드).

## 이벤트

```ts
grid.on("rowDetailExpand", ({ id, row, expanded }) => {
  // 행 ID 문자열, 행 객체, 펼침 여부
});
```

행 객체가 현재 데이터에 없을 때(append 대기 등)는 ID만 저장되고 이벤트는
발행되지 않는다.

## 스타일링

| 클래스               | 대상                     |
| -------------------- | ------------------------ |
| `.mg-detail-toggle`  | 첫 셀의 ▸/▾ 버튼         |
| `.mg-detail-row`     | 상세 `<tr>`              |
| `.mg-detail-cell`    | 전체 너비 `<td>`         |
| `.mg-detail-content` | 내용 래퍼 (기본 padding) |

`--grid-detail-bg-color` 변수로 상세 행 배경을 바꿀 수 있다.

## vanilla `detailRenderer` 반환값

- `Node` — 그대로 삽입 (DOM API로 안전하게 구성 권장)
- `string` — `textContent`로 삽입 (HTML이 아니라 텍스트 — XSS 안전)
- `null`/`undefined` — 빈 패널

## 주의 사항

- 그룹 헤더·소계 행에는 상세가 붙지 않는다 — 리프(데이터) 행 전용이다.
- 가상 스크롤·페이징과 함께 쓸 수 있다 — 상세 행 높이는 내용에 따라
  달라지므로 균일 높이가 필요하면 `detailHeight`를 지정한다.
- 상세 패널 안에서 중첩 DataGrid를 렌더링하는 것도 가능하다 (슬롯 안에
  컴포넌트 배치).
