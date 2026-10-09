# 행 드래그앤드롭 (Row Reordering)

드래그 핸들로 행 순서를 재정렬한다. 코어는 순서 이동과 드래그 상태를 관리하고
어댑터가 HTML5 DnD 이벤트를 연결한다.

## 설정

```ts
{ field: "order", header: "", width: 40, rowDrag: true }  // 이 컬럼 셀에 핸들
```

## 사용자 조작 (DataGrid 기본 동작)

- 핸들 셀(`.mg-row-drag-handle`, `⠿` 아이콘, HTML5 `draggable`)을 드래그.
- 드래그 중 행은 `.mg-row-dragging`(반투명), 드롭 갭 위치는
  `.mg-drop-before`/`.mg-drop-after` 파란 가이드 라인으로 표시.
- `Escape` 또는 `dragend`로 취소.
- 정렬·그룹화·서버 모드에서는 핸들이 비활성(`.mg-disabled`)으로 표시
  — 표시 순서가 rawData와 무관하기 때문이다. **트리 모드는 예외**로
  계층을 바꾸는 3방향 드롭이 지원된다 (아래 참조).

## 코어 API

```ts
// 직접 이동 + 이벤트
grid.moveRow(fromIndex, toIndex); // visibleData 기준, 성공 시 true
grid.on("rowReorder", (e) => {
  // { fromIndex, toIndex, row }
  saveOrder(e); // 서버 저장 등
});

// 드래그 라이프사이클 (어댑터가 HTML5 DnD에 연결)
grid.isRowDraggable(); // 가능 여부
grid.beginRowDrag(rowIndex); // 시작 (불가/범위 밖이면 false)
grid.updateRowDropPosition(gapIndex); // i면 i번 행 위, 행 수면 마지막 아래
grid.endRowDrag(true); // 커밋 → moveRow / false면 취소
grid.getRowDragState(); // { draggingIndex, dropIndex } | null
```

## 그리드 간 행 드래그 (rowDragAcceptExternal)

`rowDragAcceptExternal: true`인 그리드는 **다른 그리드에서** 드래그한
행을 드롭으로 받는다 — 이동(move) 시맨틱으로, 소스 그리드에서는
드롭된 행이 제거된다.

```ts
const source = new GridCore({ columns, data: srcRows }); // rowDrag 컬럼 필요
const target = new GridCore({ columns, data: [], rowDragAcceptExternal: true });
```

- 어댑터 prop: `<DataGrid rowDragAcceptExternal />` (React/Vue/
  Vue2/Svelte), `mountGrid({ rowDragAcceptExternal: true })` (vanilla)
- 소스에는 아무 옵션도 필요 없다 — 컬럼의 `rowDrag` 핸들로 시작된
  드래그는 자동으로 외부 세션이 된다. 대상에도 `rowDrag` 컬럼이 있으면
  양방향 이동이 된다.
- 대상 행 위로 `dragover`하면 포인터 위쪽/아래쪽 절반으로 삽입 갭을
  계산해 `.mg-drop-before`/`.mg-drop-after` 인디케이터를 표시한다.
- `drop` 시 대상의 `insertIndex` 위치에 행이 삽입되고, 성공하면 소스
  행이 제거된다. 소스의 `dragend`(Escape/취소)는 세션과 인디케이터를
  정리한다.
- 서버사이드(`serverSide`) 그리드는 대상이 될 수 없다 — 원격 행은
  로컬 삽입이 불가능하기 때문이다.
- 소스/대상의 행 타입이 달라도 동작한다 (드래그된 행 객체가 그대로
  이전된다 — 필요하면 `externalRowDrop`에서 변환 후 직접 삽입).

### 이벤트

```ts
// 대상 그리드 — 드롭으로 들어온 행
target.on("externalRowDrop", (e) => {
  // { rows: TData[], insertIndex, sourceGrid }
});

// 소스 그리드 — 드롭으로 나간 행 (성공 커밋 시에만)
source.on("externalRowRemove", (e) => {
  // { rows: TData[], targetGrid }
});
```

### 코어 API

```ts
grid.canAcceptExternalRowDrop(); // 지금 외부 드래그를 받을 수 있는가
grid.setRowDragAcceptExternal(true); // 런타임 토글 (prop 동기화 경로)
```

## 트리 모드 행 드래그 (3방향 드롭)

`treeData` 모드에서는 행 드래그가 **계층 편집**으로 동작한다 — 어댑터가
대상 행의 포인터 Y를 세 구간으로 나눠 위치를 결정한다:

| 포인터 위치   | position   | 의미                              |
| ------------- | ---------- | --------------------------------- |
| 행 위쪽 25%   | `"before"` | 대상과 같은 부모의 바로 위 형제   |
| 행 가운데 50% | `"inside"` | 대상의 자식으로 (리파렌팅, 막내)  |
| 행 아래쪽 25% | `"after"`  | 대상과 같은 부모의 바로 아래 형제 |

인디케이터: before/after는 경계선(`.mg-drop-before`/`.mg-drop-after`),
inside는 행 전체 테두리(`.mg-drop-inside`), 거부된 위치는 빨간 스타일
(`.mg-drop-denied`).

```ts
new GridCore({
  columns, // rowDrag: true 컬럼 필요
  data,
  treeData: {
    getParentId: (r) => r.parentId,
    // flat 모드에서 부모가 바뀌는 드롭을 허용하려면 필수 — 없으면
    // 부모 변경 드롭은 거부된다 (같은 부모 내 순서 변경은 가능)
    setParentId: (r, parent) => {
      r.parentId = parent ? parent.id : null;
    },
    // 개발자 가드 — false 또는 사유 문자열 반환 시 해당 위치 거부.
    // 일반 사용자에게 "왜 안 되는지" 보여줄 때 문자열을 반환한다
    canDropRow: ({ row, targetRow, position, newParentRow }) =>
      position === "inside" && newParentRow?.locked
        ? "잠긴 노드 아래로는 이동할 수 없습니다"
        : true,
  },
});
```

- **nested 모드**(`childrenKey`)는 `setParentId` 없이도 리파렌팅된다 —
  소스/대상의 자식 배열을 직접 조작한다.
- 자기 자신 또는 자손 안으로의 드롭(사이클)은 항상 거부된다.
- inside 드롭이 커밋되면 대상 노드는 자동으로 펼쳐진다.
- 서버사이드 또는 정렬 중에는 트리 드래그도 비활성이다
  (`grid.isTreeRowDraggable()`).

```ts
// 이동 성공 — 평면과 같은 rowReorder에 트리 정보가 추가된다
grid.on("rowReorder", (e) => {
  // { fromIndex, toIndex, row, position, newParentRow }
});

// 거부된 드롭 — 데이터 변경 없이 사유를 알린다
grid.on("rowDropDenied", (e) => {
  // { row, targetRow, position, newParentRow, reason }
  toast(e.reason ?? "이 위치에는 놓을 수 없습니다");
});
```

## 동작 규칙

- 인덱스는 **visibleData 기준** — 필터가 걸려 있어도 화면에 보이는 순서
  관계가 유지되도록 rawData에서 목표 행 기준으로 재삽입한다.
- `rowReorder` 이벤트는 데이터 파이프라인 재계산 후 발행된다.
- `refresh()`(데이터 변경) 시 진행 중 드래그는 자동 해제된다.
- 스냅샷 `rowDrag`로 진행 상태를 노출한다.
