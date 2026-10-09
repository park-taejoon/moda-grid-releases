# 트리 데이터 (Tree Data)

부모-자식 계층 데이터를 펼침/접힘 트리로 렌더링한다.

## 설정 — 두 가지 모드

```ts
// flat 모드 — data는 평면 배열, parentId로 연결
new GridCore({
  columns,
  data,
  getRowId: (r) => r.id, // 노드 키의 기준
  treeData: { getParentId: (r) => r.parentId },
});

// nested 모드 — 행 객체 안의 자식 배열 (기본 키 'children')
new GridCore({
  columns,
  data,
  treeData: { childrenKey: "children" },
});
```

어댑터에서는 `treeData` prop으로 전달 (내부 생성 모드) 또는
`useGridCore`/`useGrid`/`createGridStore` 옵션으로 전달.

## 코어 API

```ts
grid.toggleTreeExpanded(rowId); // 노드 펼침/접힘
grid.expandAllGroups(); // 전체 펼치기 (그룹화와 공유)
grid.collapseAllGroups();

// 레벨 펼침/접기 (타사 그리드 레벨 접기)
grid.setTreeExpandLevel(0); // 전체 접기 (루트만 표시)
grid.setTreeExpandLevel(1); // 루트 펼침 → 자식까지 표시
grid.getTreeMaxDepth(); // 최대 깊이 (루트=0)

// 노드 검색 — 조건 행의 상위를 모두 펼치고 활성 셀로 이동
grid.revealTreeRow((row) => row.name === "Frontend"); // 찾으면 true

// 트리 소계 — 부모 노드의 자손 리프 집계 (aggregationFn 컬럼별)
grid.getTreeAggregates(row); // { headcount: 16 } 또는 null (자식 없음)
```

## 사용자 조작 (DataGrid 기본 동작)

- 첫 컬럼 셀에 `hasChildren` 노드의 토글 버튼(`.mg-tree-toggle`, `▾/▸`)이
  렌더링되고 클릭 시 `toggleTreeExpanded(id)` 호출.
- 리프 행은 `depth`만큼 들여쓰기. 자식 없는 노드는 `.mg-tree-leaf`
  스페이서로 정렬.

## 자손 연쇄 선택 (`selectsChildren`)

`treeData.selectsChildren: true`이면 행 체크박스(`rowCheckboxes`)가
부모-자식 연쇄로 동작한다.

```ts
new GridCore({
  columns,
  data: treeRows,
  treeData: { getParentId: (r) => r.parentId, selectsChildren: true },
});
```

- 부모 행의 체크박스를 토글하면 **자신+모든 자손**이 함께 선택/해제된다.
  자손 일부만 선택된 상태에서 클릭하면 전체 선택이 된다.
- 자손 일부만 선택된 부모의 체크박스는 `indeterminate`(삼중 상태)로
  표시된다. 집계 상태는 `grid.treeRowSelectState(id)` →
  `"all" | "some" | "none"`으로도 조회할 수 있다.
- `isRowSelectable`로 거부된 행은 연쇄 대상과 집계 모두에서 제외된다.
- flat/nested 두 모드 모두 지원한다. 행 클릭 선택(`selectable`)도 같은
  규칙으로 연쇄된다.

## 동작 규칙

- `snapshot.displayRows`에 DFS 순서의 `LeafDisplayRow[]`가 들어간다 — 각
  리프에 `depth`, `hasChildren`, `expanded`가 부여되고 접힌 노드의 하위는
  목록에서 제외.
- 펼침 상태는 `expandedRowKeys`(행 ID 기반, 그룹화와 공유)에 저장 —
  새 노드는 기본 펼침, 접은 노드는 `setData` 후에도 접힘 유지.
- **flat 모드**: 필터/정렬 파이프라인이 끝난 `visibleData` 안에서 parentId로
  재연결. 부모가 필터링된 고아 행은 루트로 승격. 정렬은 부모 내 자식의
  상대 순서를 유지한 채 루트/형제 순서에만 반영. parentId 사이클은 감지되어
  잔여 행을 루트로 표시.
- **nested 모드**: 자식은 로컬 파이프라인을 거치지 않는다 (루트만 필터/정렬
  대상). DFS 평탄 목록으로 `visibleData`를 교체해 선택/편집의 `rowIndex`가
  자식 행에서도 일치.
- `displayRows` 길이가 가상 스크롤 `totalHeight`를 결정 — 접힘/펼침이
  스크롤에 즉시 반영.
- 트리 모드에서는 `groupBy`가 비활성화된다. 행 드래그는 3방향 트리
  드롭으로 동작한다 — `row-drag.md`의 "트리 모드 행 드래그" 참조.

## 행 드래그로 계층 편집 (`setParentId` / `canDropRow`)

`rowDrag: true` 컬럼의 핸들로 노드를 드래그하면 대상 행 위/아래(형제
순서) 또는 안쪽(자식으로)에 드롭할 수 있다. 자세한 규칙은
[행 드래그앤드롭](./row-drag.md#트리-모드-행-드래그-3방향-드롭) 참조.

```ts
treeData: {
  getParentId: (r) => r.parentId,
  // flat 모드에서 부모를 바꾸는 드롭을 허용 — 새 부모를 행에 써넣는다
  setParentId: (row, parent) => { row.parentId = parent?.id ?? null; },
  // 거부 규칙 — false 또는 사유 문자열 (→ rowDropDenied 이벤트)
  canDropRow: ({ position, newParentRow }) =>
    position === "inside" && newParentRow == null
      ? "최상위로는 이동할 수 없습니다"
      : true,
}
```

- flat 모드에서 `setParentId`가 없으면 같은 부모 내 순서 변경만
  허용되고 부모 변경 드롭은 거부된다.
- nested 모드(`childrenKey`)는 자식 배열을 직접 조작하므로
  `setParentId` 없이 리파렌팅된다.
