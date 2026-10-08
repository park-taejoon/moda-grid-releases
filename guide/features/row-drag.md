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
- 정렬·그룹화·트리·서버 모드에서는 핸들이 비활성(`.mg-disabled`)으로 표시
  — 표시 순서가 rawData와 무관하기 때문이다.

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

## 동작 규칙

- 인덱스는 **visibleData 기준** — 필터가 걸려 있어도 화면에 보이는 순서
  관계가 유지되도록 rawData에서 목표 행 기준으로 재삽입한다.
- `rowReorder` 이벤트는 데이터 파이프라인 재계산 후 발행된다.
- `refresh()`(데이터 변경) 시 진행 중 드래그는 자동 해제된다.
- 스냅샷 `rowDrag`로 진행 상태를 노출한다.
