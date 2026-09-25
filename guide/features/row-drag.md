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
grid.moveRow(fromIndex, toIndex);              // visibleData 기준, 성공 시 true
grid.on("rowReorder", (e) => {                 // { fromIndex, toIndex, row }
  saveOrder(e);                                // 서버 저장 등
});

// 드래그 라이프사이클 (어댑터가 HTML5 DnD에 연결)
grid.isRowDraggable();                         // 가능 여부
grid.beginRowDrag(rowIndex);                   // 시작 (불가/범위 밖이면 false)
grid.updateRowDropPosition(gapIndex);          // i면 i번 행 위, 행 수면 마지막 아래
grid.endRowDrag(true);                         // 커밋 → moveRow / false면 취소
grid.getRowDragState();                        // { draggingIndex, dropIndex } | null
```

## 동작 규칙

- 인덱스는 **visibleData 기준** — 필터가 걸려 있어도 화면에 보이는 순서
  관계가 유지되도록 rawData에서 목표 행 기준으로 재삽입한다.
- `rowReorder` 이벤트는 데이터 파이프라인 재계산 후 발행된다.
- `refresh()`(데이터 변경) 시 진행 중 드래그는 자동 해제된다.
- 스냅샷 `rowDrag`로 진행 상태를 노출한다.
