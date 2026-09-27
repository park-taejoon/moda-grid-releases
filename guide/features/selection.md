# 선택 (Selection)

행 클릭 선택, 셀/범위 선택, 행 체크박스, 선택 영역 집계 상태바를 제공한다.

## 선택 모드

`selectionMode` prop(또는 `GridOptions.selectionMode`):

| 모드 | 동작 |
| ---- | ---- |
| `single-cell` (기본) | 활성 셀 하나만. 범위 선택 없음 |
| `multi-cell` | Shift+이동/드래그로 사각 범위 선택 |
| `row` | 활성 셀 대신 행 전체가 선택되고 Shift로 행 단위 확장 |

## 사용자 조작 (DataGrid 기본 동작)

| 조작 | 동작 |
| ---- | ---- |
| 셀 클릭 | 활성 셀 지정 (`.mg-cell-active` 아웃라인) |
| 셀 드래그 | 누른 셀을 앵커로 드래그 중인 셀까지 범위 선택 — `multi-cell`/`row` 모드 |
| 방향키 | 인접 셀로 이동 (경계 클램프) |
| Home / End | 행의 첫/마지막 컬럼으로 이동 |
| PageUp / PageDown | 뷰포트 행 수 단위 이동 |
| Shift + 위 키 | 범위 확장 (`.mg-cell-selected`) — `multi-cell`/`row` 모드 |
| Tab / Shift+Tab | 활성 셀 좌/우 이동 — 행 끝에서 다음/이전 행으로 wrap (엑셀) |
| Home / End | 행의 첫/마지막 셀로 이동 |
| PageUp / PageDown | 뷰포트 행 수만큼 위/아래 이동 |
| Ctrl/Cmd + Home / End | 그리드 첫/마지막 셀로 이동 (`"first"`/`"last"`) |
| Ctrl/Cmd + ↑ / ↓ / ← / → | 같은 열 첫/마지막 행, 같은 행 첫/마지막 열로 이동 |
| Shift + Space | 행 전체 선택 (`selectEntireRow`, `single-cell`은 행 선택으로 대체) |
| Ctrl/Cmd + Space | 컬럼 전체 선택 (`selectEntireColumn`, `multi-cell`만) |
| Ctrl/Cmd + D | 선택 범위 첫 행을 아래로 채우기 (`fillDown`) |
| Ctrl/Cmd + R | 선택 범위 첫 컬럼을 오른쪽으로 채우기 (`fillRight`) |
| Ctrl/Cmd + X | 잘라내기 — 복사 + 내용 지우기, Undo 1단위 (`cutSelectionTsv`) |
| Ctrl/Cmd + Enter (편집 중) | 입력 값을 선택 범위 전체에 적용 (`fillActiveToSelection`) |
| Ctrl/Cmd + A | 전체 선택 — `multi-cell`/`row`는 전체 범위, `single-cell`은 전체 행 |
| Escape | 선택 해제 |
| 행 클릭 | `toggleRowSelection` — `selectable` prop으로 비활성 가능 |
| Tab | 편집 종료 시 이동 ([editing.md](./editing.md) 참고) |

활성 셀이 가상 스크롤 뷰포트 밖이면 코어가 `virtual.scrollTop`을 자동 보정해
DOM 스크롤과 동기화한다 (scrollIntoView).

## 코어 API

```ts
// 셀/범위
grid.setSelectionMode("multi-cell");
grid.setActiveCell(rowIndex, columnIndex);      // visibleData/visibleColumns 기준
grid.setCellRange({ startRow: 0, startCol: 0, endRow: 3, endCol: 2 });
grid.extendSelectionTo(rowIndex, columnIndex); // 앵커 유지, 활성 셀만 이동 (드래그)
grid.navigateCell("down", true);                // dir + extend(Shift)
grid.selectAll();                               // Ctrl+A와 동일
grid.selectEntireRow(); grid.selectEntireColumn(); // Shift+Space / Ctrl+Space
grid.fillDown(); grid.fillRight();              // Ctrl+D / Ctrl+R — 선택 범위 채우기
grid.cutSelectionTsv();                         // Ctrl+X — TSV 반환 + 지우기
grid.clearCellSelection();
grid.isActiveCell(r, c); grid.isCellInRange(r, c);

// 행 선택
grid.toggleRowSelection(id);
grid.clearSelection();
grid.isSelected(id);
grid.toggleAllRows();                           // 전체 토글 (헤더 체크박스)
grid.isAllSelected(); grid.isSomeSelected();
```

## 행 체크박스 (`rowCheckboxes` prop)

`rowCheckboxes` prop을 주면 맨 앞에 체크박스 컬럼(`.mg-check-cell`)이 생긴다:
헤더는 전체선택(불확정 상태 지원), 각 행은 개별 체크박스.
그룹/스켈레톤/고정 행에는 빈 셀로 정렬만 맞춘다.

## 선택 영역 집계 (상태바)

선택 범위(없으면 활성 셀)가 있으면 스냅샷에 집계가 들어간다:

```ts
snapshot.selectionAggregates = {
  cells: 6, count: 4, sum: 152, avg: 38, min: 12, max: 90,
};
// cells: 범위 내 셀 수, count: 숫자 해석 가능 셀 수, 숫자 없으면 null
```

`statusBar` prop(기본값 `true`)으로 그리드 하단에 자동 표시된다
(`.mg-statusbar`).

## 스냅샷 필드

`snapshot.activeCell` / `snapshot.selectedRange` / `snapshot.selectionMode` /
`snapshot.selectedRowIds` / `snapshot.selectionAggregates`.

선택된 행의 **데이터**가 필요하면 `grid.getSelectedRowData()`를 사용한다 —
숨김·필터로 화면에서 빠진 선택 행도 포함되고, 순서는 rawData 순이다
(타사 그리드 `FindCheckedRow`/`GetRowsByStatus` 계열 대응).

## 주의사항

- 인덱스는 **visibleData/visibleColumns 기준** — 정렬·필터·페이징·컬럼
  재배치 후의 상대 위치다. 그룹 리프 행의 `rowIndex`도 같은 기준이라
  호환된다.
- input/textarea/select 안의 키 입력은 그리드 네비게이션과 분리된다
  (에디터 내부 키는 `stopPropagation` 처리됨).
