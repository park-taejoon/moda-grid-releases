# @moda-grid/core — GridCore API

프레임워크에 **전혀 의존하지 않는** 순수 TypeScript 그리드 코어 클래스.
DOM을 모르며, 정렬·컬럼 필터·전역 검색·페이징·행 선택 같은 상태와
파생 데이터 계산만 담당한다. `grid.test.ts`에 vitest 기반 테스트가 있다.

## GridCore 생성

```ts
import { GridCore } from "@moda-grid/core";
// 팩토리 함수도 제공: createGrid(options) === new GridCore(options)

interface User { id: number; name: string; age: number; }

const grid = new GridCore<User>({
  columns: [
    { field: "id", header: "ID", width: 60 },
    { field: "name", header: "이름", filterable: true },
    { field: "age", header: "나이" },
  ],
  data: users,
  getRowId: (row) => String(row.id), // 생략 시 row.id → WeakMap 자동 ID
  pageSize: 20,                      // 초기 페이지 크기 (기본값 0 = 비활성)
  // 선택: serverSide(무한 스크롤), treeData(계층 데이터),
  //       selectionMode, virtualScroll — 각 섹션 참고
});
```

## 핵심 데이터 필드

| 필드 | 타입 | 설명 |
| ---- | ---- | ---- |
| `rawData` | `TData[]` | 원본 데이터 (`setData`로 교체) |
| `visibleData` | `TData[]` | **필터 → 검색 → 정렬 → 페이징 적용 후 출력 데이터** |
| `columns` | `ColumnDef[]` | 컬럼 정의 배열 (`field`/`header`/`width`/`sortable`/`filterable` 등) |

`visibleData`는 매번 `notify()` 시점에 파이프라인 전체를 다시 계산해
공개 필드로 갱신된다.

## Pub/Sub 이벤트 시스템

```ts
const unsubscribe = grid.subscribe(() => {
  console.log(grid.visibleData.length, "rows");
});
grid.notify();   // 수동으로 재계산 + 알림 발행
unsubscribe();   // 구독 해제
grid.destroy();  // 리스너 전체 해제
```

- 모든 뮤테이션 메서드(`setData`, `setSort`, `setFilter`, `setPage` …)는
  내부적으로 `notify()`를 호출하므로 보통 수동 호출은 불필요하다.
- `notify()`는 ① 파생 데이터 재계산(`refresh`) → ② 모든 리스너 호출 순으로
  동작한다.
- `subscribe` 외에 개별 액션 단위의 **타입 이벤트**는 `grid.on(event, cb)`로
  구독한다 — 현재 `rowReorder`(`{ fromIndex, toIndex, row }`)를 지원한다.

## 주요 기능 API

| 메서드 | 시그니처 | 동작 |
| ------ | -------- | ---- |
| `setData` | `(data: TData[])` | rawData 교체 (방어적 복사) |
| `setSort` | `(field, direction: "asc" \| "desc" \| null)` | 단일 정렬 지정/해제 (기존 조건 전부 교체) |
| `setFilter` | `(field, value: string \| ColumnFilter \| null \| undefined)` | 컬럼별 필터. 문자열이면 contains, 객체면 연산자 조건. 빈 값이면 해제, pageIndex 0 리셋 |
| `setPage` | `(pageIndex: number, pageSize?: number)` | 페이지 이동. `size=0`이면 페이징 비활성 |

## 반응형 어댑터 계약 (getSnapshot)

프레임워크 어댑터는 `subscribe` + `getSnapshot()` 조합을 쓴다.
스냅샷은 **메모이즈**되어 변경 없으면 같은 참조를 반환한다
(React `useSyncExternalStore` 필수 요건).

```ts
grid.subscribe(() => render(grid.getSnapshot()));
```

`GridSnapshot` 필드:

| 필드 | 설명 |
| ---- | ---- |
| `columns` / `visibleColumns` | 전체 / 표시 컬럼 |
| `rows` | `visibleData`와 동일한 최종 출력 행 |
| `totalRowCount` / `filteredRowCount` | 필터 전 전체 수 / 페이징 전 수 |
| `sort` / `sortState` | 첫 번째 정렬 조건(호환) / 다중 정렬 조건 `SortSpec[]` |
| `groupBy` / `expandedRowKeys` | 그룹화 필드 목록 / 펼친 그룹 키 집합 |
| `displayRows` | 그룹화 시 평탄 렌더 목록 `DisplayRow[]` (아니면 `null`) |
| `searchText` / `filters` | 전역 검색어 / 컬럼 필터 `Record<field, ColumnFilter>` |
| `pageIndex` / `pageSize` / `pageCount` | 페이징 상태 |
| `selectedRowIds` | 선택된 행 ID 집합 |
| `rowDrag` | 행 드래그 진행 상태 `{ draggingIndex, dropIndex }` (아니면 `null`) |
| `pinnedTopRows` / `pinnedBottomRows` | 상단/하단 고정 행 데이터 (`TData[]`) |
| `grandTotals` | `aggregationFn` 컬럼의 전체 집계 (해당 컬럼 없으면 `null`) |
| `selectionAggregates` | 선택 범위 집계 `{ cells, count, sum, avg, min, max }` (선택 없으면 `null`) |
| `headerGroups` | 상단 그룹 헤더 스팬 `HeaderGroupSpan[]` (`ColumnDef.group` 없으면 `null`) |

## 확장 API (어댑터·앱 공용)

| 메서드 | 동작 |
| ------ | ---- |
| `setColumns(cols)` | 컬럼 정의 교체 |
| `toggleSort(field, additive?)` | `asc → desc → 해제` 순환. `additive=true`(Shift+클릭)이면 다중 정렬 조건에 추가/제거 |
| `clearSorts()` | 모든 정렬 조건 해제 |
| `setGroupBy(fields \| null)` | 행 그룹화 기준 필드 목록 설정 (`['dept','role']`) |
| `toggleGroupExpanded(key)` | 그룹 행 펼침/접힘 토글 |
| `expandAllGroups()` / `collapseAllGroups()` | 전체 펼치기/접기 |
| `isGroupExpanded(key)` | 그룹 키 펼침 여부 |
| `exportToCsv(options?)` | 표시 상태 데이터를 CSV 문자열로 반환 + 브라우저면 다운로드 |
| `getSelectionTsv(options?)` | 선택 범위(없으면 활성 셀)를 TSV 문자열로 변환 (없으면 null). `{ includeHeaders: true }`면 첫 줄에 헤더 행 포함 |
| `pasteTsv(tsv, start?)` | TSV를 활성 셀부터 순차 쓰기 → `PasteResult` 반환 (편집 이력에 1개 단위로 기록) |
| `setSearch(text)` | 전역 검색 (표시 컬럼 전체, 대소문자 무시) |
| `clearFilters()` | 모든 컬럼 필터 해제 |
| `toggleRowSelection(id)` / `clearSelection()` / `isSelected(id)` | 행 선택 |
| `toggleAllRows()` / `isAllSelected()` / `isSomeSelected()` | 전체 행 선택 토글 / 전체·일부 선택 여부 (헤더 체크박스용) |
| `setColumnVisibility(field, visible)` / `setColumnVisible(columnId, visible)` | 컬럼 표시/숨김 (동일 동작 — columnId는 `field`와 같음) |
| `setAllColumnsVisible(visible)` | 전체 컬럼 일괄 표시/숨김 (Column Controller의 전체 선택/해제) |
| `setColumnWidth(field, width)` | 컬럼 너비 변경 (minWidth/maxWidth 클램프) |
| `autoSizeColumn(field, measureText?)` / `autoSizeAllColumns(measureText?)` | 내용 기준 자동 너비. `measureText` 미지정 시 문자 길이 기반 추정 (DOM 없이 동작) |
| `reorderColumn(draggedId, targetId)` | 컬럼 순서 변경 (숨김 포함 전체 순서 기준) |
| `resetColumnLayout()` | 너비/순서를 컬럼 정의 기본값으로 초기화 |
| `setColumnPinned(field, pinned)` | 컬럼 고정 위치 변경 (`'left'`/`'right'`/`null`) |
| `isServerSide()` / `isRowLoaded(i)` | 서버 모드 여부 / 해당 인덱스 로드 여부 (스켈레톤 판별) |
| `refreshServerRows()` | 서버 캐시 폐기 + 현재 뷰포트부터 재요청 |
| `moveRow(from, to)` | 행 순서 이동 + `rowReorder` 이벤트 발행 |
| `beginRowDrag` / `updateRowDropPosition` / `endRowDrag` | 행 드래그 라이프사이클 (어댑터 연결용) |
| `on(event, cb)` | 타입 이벤트 구독 (`rowReorder` 등) |
| `setSelectionMode(mode)` | 셀 선택 모드 (`'single-cell'`/`'multi-cell'`/`'row'`) |
| `setActiveCell(row, col)` | 활성 셀 지정 (범위 해제 + 클램프 + scrollIntoView) |
| `setCellRange(range \| null)` | 선택 범위 직접 지정 (정규화 적용) |
| `navigateCell(dir, extend?)` | 키보드 셀 네비게이션 (아래 셀 선택 참고) |
| `clearCellSelection()` | 활성 셀/범위 해제 |
| `isActiveCell(r, c)` / `isCellInRange(r, c)` | 렌더링 시 셀 상태 조회 |
| `getScrollTop()` | 현재 스크롤 위치 (DOM 동기화용) |
| `startEditing(row, col, initial?)` | 편집 진입 (`editable: false`면 false) |
| `updateEditValue(v)` | 임시 입력 값 갱신 (에러 해제 포함) |
| `commitEditing()` | 검증 후 rawData에 저장. 실패 시 false + 편집 유지 |
| `cancelEditing()` | 저장 없이 편집 종료 |
| `isEditing(r, c)` / `isCellEditable(r, c)` | 렌더링 시 편집 상태 조회 |
| `getEditorContext(r, c)` | 커스텀 편집기용 컨텍스트 (`CellEditorContext`) |
| `undo()` / `redo()` / `canUndo()` / `canRedo()` | 편집·붙여넣기 이력 되돌리기/다시 실행 (아래 Undo/Redo 참고) |
| `setPinnedTopRows(rows)` / `setPinnedBottomRows(rows)` | 상단/하단 고정 행 데이터 설정 (`null`로 해제) |
| `getUniqueValues(field)` | 컬럼의 고유 표시 값 목록 — Set 필터 체크리스트용 |
| `getState()` / `applyState(state)` | 직렬화 가능한 그리드 상태 저장/복원 (`GridPersistedState`) |
| `getRowId(row)` | 행 고유 ID (렌더 key용) |
| `getCellText(row, col)` | `formatter`/`valueGetter` 적용된 표시 문자열 |
| `getCellClass(row, rowIndex, col)` / `getRowClass(row, rowIndex)` | `cellClass`/`rowClass` 해석된 커스텀 클래스 문자열 |

## ColumnDef

```ts
interface ColumnDef<TData> {
  field: keyof TData & string;             // 컬럼 키 (명세의 key)
  header?: string;                         // 헤더 텍스트 (명세의 title)
  width?: number;
  minWidth?: number;                       // 리사이즈 최소 (기본값 DEFAULT_MIN_COLUMN_WIDTH=40)
  maxWidth?: number;                       // 리사이즈 최대 (기본값 무제한)
  resizable?: boolean;                     // 기본값 true
  sortable?: boolean;                      // 기본값 true
  visible?: boolean;                       // 기본값 true
  pinned?: 'left' | 'right' | null;        // 좌/우 고정 (기본값: 고정 없음)
  filterable?: boolean;                    // 기본값 false — 헤더 필터 입력 표시
  filterType?: 'text'|'number'|'date'|'set'; // 기본값 'text' — 연산자 목록 결정 ('set'은 값 체크리스트)
  group?: string;                          // 컬럼 그룹 ID — 다단계 헤더 (GridOptions.columnGroups 참고)
  comparator?: (a: TData, b: TData) => number;
  valueGetter?: (row: TData) => unknown;
  formatter?: (value: unknown, row: TData) => string;
  filterPredicate?: (value, row, filter: ColumnFilter) => boolean; // 커스텀 필터 판별
  aggregationFn?: 'sum'|'avg'|'min'|'max'|'count'; // 그룹 행 집계
  editable?: boolean;                      // 기본값 true — 인라인 편집 허용
  cellEditor?: 'text'|'number'|'select'|'date'|'custom'; // 기본값 'text'
  editorOptions?: readonly string[];       // select 편집기 옵션
  valueSetter?: (row, value) => void;      // 기본값: row[field] = value
  validate?: (value, row) => boolean | string; // 편집 저장 전 검증
  rowDrag?: boolean;                       // 이 컬럼 셀에 행 드래그 핸들 표시
  cellClass?: ClassSource<CellClassParams>; // 셀 커스텀 클래스 (문자열 | 함수)
  rowClass?: ClassSource<RowClassParams>;   // 행 커스텀 클래스 — 여러 컬럼이 합산
}
```

- 기본 정렬: `Intl.Collator(numeric: true)` — 숫자/날짜/문자열 자연 비교
- 기본 필터: `filter.operator` 기반 내장 평가 (`matchFilterValue`),
  `filterPredicate`로 컬럼별 교체 가능
- 복수 컬럼 필터는 AND로 결합된다

## 다중 정렬 (Multi-Sort)

`sortState`는 `SortSpec[]` — `{ columnKey, direction, priority }` 형태로
관리되며, `priority` 순으로 체인 비교한다 (앞 조건이 같으면 다음 조건).

```ts
grid.toggleSort("role");          // 일반 클릭: 단일 정렬 (기존 조건 교체)
grid.toggleSort("age", true);     // Shift+클릭: 조건 추가 → priority 2
grid.toggleSort("age", true);     // 다시 Shift+클릭: desc
grid.toggleSort("age", true);     // 한 번 더: 조건 제거, priority 재정규화
```

- `setSort(field, dir)`는 모든 조건을 단일 조건으로 교체한다.
- `clearSorts()`는 전부 해제한다.
- 스냅샷의 `sort`는 첫 번째 조건을 그대로 노출해 단일 정렬 코드와 호환된다.

## 고급 필터 (연산자 기반)

`setFilter`는 `ColumnFilter` 객체 — `{ operator, value, valueTo? }` — 를 받는다.
평가는 순수 함수 `matchFilterValue(value, filter)`(`filtering.ts`)가 담당한다.

| 종류 | 연산자 | 의미 |
| ---- | ------ | ---- |
| text | `contains` `equals` `startsWith` `endsWith` | 문자열 비교 (대소문자 무시) |
| number | `equals` `greaterThan` `lessThan` `inRange` | `Number()` 변환 비교, `inRange`는 `value~valueTo` |
| date | `equals` `before` `after` | 날짜 파싱 후 "일" 단위 비교 (`equals`=같은 날) |
| set | `set` | `values` 배열에 포함된 값만 통과 — 엑셀식 체크박스 필터 |

- `equals`는 스마트 판별: 양쪽이 숫자면 숫자 비교, 양쪽이 날짜면 같은 날
  비교, 아니면 문자열 동일 비교.
- `value`/`valueTo`가 모두 빈 조건은 항상 통과 — 연산자만 선택된 상태에서
  모든 행이 사라지지 않는다.
- `FILTER_OPERATORS_BY_TYPE` / `DEFAULT_OPERATOR_BY_TYPE` /
  `FILTER_OPERATOR_LABELS` 상수로 타입별 연산자 목록을 조회할 수 있다
  (어댑터의 연산자 셀렉트가 사용).

### Set 필터 (값 목록 필터)

`filterType: 'set'` 컬럼은 연산자 입력 UI 대신 **값 체크리스트**로 필터링한다:

```ts
{ field: "role", header: "직책", filterable: true, filterType: "set" }

grid.getUniqueValues("role");   // ["admin", "user", ...] — 체크리스트 항목
grid.setFilter("role", { operator: "set", value: "", values: ["admin"] });
grid.setFilter("role", null);   // 전체 선택 = 필터 해제
```

- 비교는 셀 값의 문자열 표현(`String(value)`, `valueGetter` 적용 값) 기준 —
  `values`에 없는 값은 모두 걸러진다.
- `values`가 고유값 전체를 포함하면 필터가 해제된 것과 동일.
- 어댑터는 `(전체 선택)` 항목 → `setFilter(field, null)`, 개별 토글 →
  현재 `values`에서 추가/제거 후 `operator: 'set'`으로 다시 지정한다.

## 행 그룹화 (Row Grouping) + 집계

`setGroupBy(['dept', 'role'])`로 파이프라인 결과(`visibleData`)를
계층 구조로 변환한다 — `grouping.ts`의 순수 함수가 담당한다.

```ts
buildGroupTree(leafRows, keys, columns, readValue) // GroupNode[] 트리
flattenGroupTree(tree, expandedRowKeys)            // DisplayRow[] 평탄화
```

- `GroupNode`: `{ type:'group', key, field, value, depth, rowCount,
  aggregates, expanded, children }` — `children`에 하위 그룹/리프가 재귀로 들어간다
- `DisplayRow` = `GroupNode | LeafDisplayRow` (리프는 `row`, `rowIndex`,
  `depth` 보유 — `rowIndex`는 `visibleData` 기준이라 기존 선택/편집 API와 호환)
- 그룹 `key`는 전체 경로를 JSON 직렬화한 고유 문자열
- `expandedRowKeys: Set<string>` — 펼친 그룹 키 집합. **새 그룹은 기본 펼침**,
  사용자가 접은 그룹은 데이터 변경 후에도 접힘 유지
- 집계는 `aggregationFn`이 선언된 컬럼만 계산해 `node.aggregates[field]`에 저장
  (`count`=행 수, 나머지는 숫자 변환 가능한 값만 합산)
- 스냅샷 `displayRows`가 null이 아니면 그룹 모드 — 어댑터는 그룹 헤더 행
  (클릭 → `toggleGroupExpanded`)과 리프 행을 섞어 렌더링한다
- 가상 스크롤과 병용: `virtual.totalHeight`는 그룹 헤더를 포함한 평탄 행 수
  기준이고, `virtualRows` 대신 `displayRows.slice(startIndex, endIndex)`를 쓴다
- 집계 값 표시는 `formatAggregate()` 헬퍼 사용 (소수 2자리 반올림)

## 트리 데이터 (계층형 데이터)

부모-자식 관계의 계층 데이터를 펼침/접힘 트리로 렌더링한다.
`GridOptions.treeData`에 두 모드 중 하나를 지정한다:

```ts
// flat 모드 — data는 평면 배열, parentId로 연결
new GridCore({
  columns, data,
  getRowId: (r) => r.id,                     // 노드 키의 기준
  treeData: { getParentId: (r) => r.parentId },
});

// nested 모드 — 행 객체 안에 자식 배열 (기본 키 'children')
new GridCore({
  columns, data,
  treeData: { childrenKey: "children" },
});
```

동작 규칙:

- **평탄화** — `displayRows`에 DFS 순서의 `LeafDisplayRow[]`가 들어간다.
  각 리프에 `depth`(들여쓰기), `hasChildren`, `expanded`가 부여되고,
  접힌 노드의 하위 트리는 목록에서 제외된다.
- **펼침 상태** — `expandedRowKeys`(그룹화와 공유)에 행 ID가 저장된다.
  새로 등장한 노드는 기본 펼침, 사용자가 접은 노드는 `setData` 후에도
  접힘 유지. `toggleTreeExpanded(rowId)`로 토글하고,
  `expandAllGroups`/`collapseAllGroups`가 트리 노드에도 동작한다.
- **flat 모드** — 필터/정렬/페이징 파이프라인이 끝난 `visibleData`
  안에서 parentId로 재연결한다. 부모가 필터링된 고아 행은 루트로 승격.
  정렬은 각 부모 내 자식의 상대 순서를 유지한 채 루트/형제 순서에만
  반영된다. parentId 사이클은 감지되어 잔여 행을 루트로 표시한다.
- **nested 모드** — 자식은 로컬 파이프라인을 거치지 않으므로(루트만
  필터/정렬 대상), DFS 평탄 목록으로 `visibleData`를 교체해 선택/편집의
  `rowIndex`가 자식 행에서도 일치한다.
- **뷰포트 연동** — `displayRows` 길이가 가상 스크롤 `totalHeight`를
  결정하므로 접힘/펼침이 스크롤 영역에 즉시 반영된다.
- 트리 모드에서는 `groupBy`와 행 드래그(`isRowDraggable() === false`)가
  비활성화된다.

| 메서드 | 동작 |
| ------ | ---- |
| `toggleTreeExpanded(rowId)` | 트리 노드 펼침/접힘 토글 |

## CSV보내기

`grid.exportToCsv(options)`는 **현재 표시 상태**(필터·검색·정렬·페이징·
그룹화가 반영된 데이터)를 CSV로 직렬화한다 (`csv.ts`).

```ts
grid.exportToCsv({
  filename: "users.csv",        // 다운로드 파일명 (기본값 "grid.csv")
  visibleColumnsOnly: true,     // 숨긴 컬럼 제외 (기본값)
  selectedRowsOnly: false,      // true면 선택된 행만 (기본값)
});
```

- 반환값: `\uFEFF`(UTF-8 BOM)가 선행된 CSV 문자열 — 엑셀에서 한글이
  깨지지 않는다. 브라우저 환경에서는 `downloadCsv`를 통해 파일
  다운로드까지 트리거하고, 비-DOM 환경(테스트/SSR)에서는 문자열만 반환
- 셀은 `formatter` 적용 표시 텍스트 기준. `,` `"` `\n` `\r` 포함 시
  큰따옴표로 감싸고 내부 `"`는 `""`로 이스케이프 (RFC 4180, CRLF 구분)
- 그룹화 중이면 펼쳐진 리프 행만 표시 순서대로 출력 (그룹 헤더 제외)
- 순수 함수 `escapeCsvCell` / `buildCsv` / `downloadCsv`도 개별 export된다

## 클립보드 (Copy & Paste)

**Copy** — `getSelectionTsv()`가 `selectedRange`(없으면 `activeCell`) 영역을
TSV로 변환한다. 어댑터가 `Ctrl/Cmd+C`에서 `navigator.clipboard.writeText()`로
저장한다. 숨김 컬럼 제외, 셀 내부 탭/줄바꿈은 공백으로 치환.
`{ includeHeaders: true }` 옵션(Ctrl+Shift+C)은 첫 줄에 헤더 행을 추가한다.

**Paste** — `pasteTsv(tsv)`가 TSV를 파싱해 `activeCell`부터 오른쪽/아래로
순차 쓰기한다:

- `editable: false` 컬럼과 범위 밖 셀은 건너뜀 (`skipped`)
- `cellEditor: 'number'` 컬럼은 숫자로 변환 (변환 불가 시 에러 기록)
- 컬럼 `validate` 통과 실패 시 건너뜀 + `errors`에 위치·메시지 기록
- 쓰기는 `valueSetter` 또는 `row[field] = value` — 편집 커밋과 동일 경로
- 반환: `PasteResult { applied, skipped, errors }`, 적용 시 1회 `notify`

어댑터는 `Ctrl/Cmd+V`에서 `navigator.clipboard.readText()`로 읽어
`grid.pasteTsv(text)`를 호출한다.

## 서버 사이드 데이터 모델 (무한 스크롤)

`GridOptions.serverSide`를 지정하면 로컬 파이프라인(정렬/필터/페이징)을
건너뛰고 필요한 행 블록을 `dataSource.getRows`로 가져온다:

```ts
interface ServerSideDataSource<TData> {
  getRows(params: {
    startRow: number;                  // inclusive
    endRow: number;                    // exclusive
    sortModel: SortSpec[];             // 현재 다중 정렬 (priority 순)
    filterModel: Record<string, ColumnFilter>;
  }): Promise<{ rows: TData[]; lastRowIndex?: number }>;
}

new GridCore({
  columns,
  data: [],                                        // 서버 모드에서는 무시
  serverSide: { dataSource, cacheBlockSize: 50 },  // 기본 블록 50행
  virtualScroll: { rowHeight, viewportHeight },    // 무한 스크롤은 가상 스크롤과 함께
});
```

동작 규칙:

- **블록 캐시** — 뷰포트가 걸치는 미로드 블록만 `getRows`로 요청하고 결과를
  `Map<rowIndex, row>`에 캐시한다. `loading`/`loaded`/`error` 상태의 블록은
  재요청하지 않는다 (실패 블록은 `refreshServerRows()`로 재시도).
- **rowCount** — `lastRowIndex`가 오면 전체 행 수가 확정된다. 없으면
  `max(요청 끝, 로드 끝) + cacheBlockSize`로 추정해 스크롤 영역을 넓히고,
  요청 크기보다 짧은 블록이 오면 데이터 끝으로 확정한다.
- **정렬/필터 변경** — `toggleSort`/`setSort`/`setFilter`/`setSearch` 등
  서버 파라미터가 바뀌면 캐시를 폐기(purge)하고 현재 뷰포트부터 재요청한다.
  purge 후 도착한 이전 세대 응답은 폐기된다 (epoch 토큰).
- **스냅샷** — `serverSide: { loading, rowCount, loadedCount, error }`.
  `rows`/`virtualRows`의 미로드 인덱스는 `undefined` — 어댑터가
  스켈레톤으로 렌더링한다.
- 서버 모드에서는 로컬 페이징과 행 그룹화가 적용되지 않는다.

## 컬럼 레이아웃 (너비/순서)

GridCore는 `columnStates: Map<field, ColumnState>`를 유지한다:

```ts
interface ColumnState {
  id: string;      // = ColumnDef.field
  width?: number;  // 현재 너비 (undefined면 자동)
  order: number;   // 표시 순서 (숨김 포함)
}
```

- 스냅샷의 `columns`/`visibleColumns`는 `order` 기준 정렬 + 상태 너비가
  병합된 결과다 — 어댑터는 그대로 렌더링하면 된다.
- `setColumns`로 정의가 교체돼도 기존 너비/순서 상태는 유지되고, 새 컬럼은
  끝에 추가, 제거된 컬럼의 상태는 삭제된다.
- `setColumnWidth`/`reorderColumn`은 데이터 파이프라인을 재계산하지 않고
  스냅샷만 갱신한다 (`invalidate`) — 리사이즈/드래그 중의 비용을 줄인다.
- 스냅샷의 `columnState: ColumnState[]`(order 오름차순)로 레이아웃 상태를
  직접 읽을 수 있다.

## 컬럼 고정 (Pinning / Freezing)

`ColumnDef.pinned`(`'left' | 'right' | null`) 또는 `setColumnPinned(field, pinned)`로
컬럼을 좌/우에 고정한다.

- **렌더링 순서** — 스냅샷 `visibleColumns`는 `[left 고정 → 비고정 → right 고정]`
  순서로 정렬된다 (각 영역 내에서는 `order` 유지). 어댑터가 그대로 렌더링하면
  DOM 순서가 `[Left Pinned | Scrollable | Right Pinned]`가 된다.
  `snapshot.columns`는 파티션 없이 `order` 기준 정렬을 유지한다.
- **오프셋** — 스냅샷 `pinOffsets: Record<field, px>`에 sticky 오프셋이 들어 있다.
  left 고정은 왼쪽부터, right 고정은 오른쪽부터 유효 너비를 누적한다
  (`width` 미지정 컬럼은 `DEFAULT_COLUMN_WIDTH`=120). 리사이즈(`setColumnWidth`)
  결과도 오프셋에 반영되고, 숨겨진 컬럼은 계산에서 제외된다.
- **인덱스 일관성** — 셀 선택/편집/클립보드의 `columnIndex`는 파티션이 적용된
  `visibleColumns` 기준이므로 DOM 렌더링 순서와 항상 일치한다.
- 어댑터는 고정 셀에 `position: sticky` + `left`/`right` 오프셋을 적용하고,
  영역 경계(`mg-pin-left-edge`/`mg-pin-right-edge`)에 구분선을 표시한다.

## 컬럼 그룹 헤더 (다단계 헤더)

`ColumnDef.group`에 그룹 ID를 지정하면 헤더가 두 단계로 렌더링된다:

```ts
new GridCore({
  columns: [
    { field: "name", header: "이름" },                          // 그룹 없음 → rowspan=2
    { field: "kor",  header: "국어", group: "score" },
    { field: "eng",  header: "영어", group: "score" },
    { field: "math", header: "수학", group: "score" },
  ],
  columnGroups: [{ id: "score", header: "성적" }],
  data,
});
```

- 스냅샷 `headerGroups: HeaderGroupSpan[] | null`에 상단 행의 병합 정보가
  계산되어 들어간다 — `{ groupId, header, startIndex, colSpan }`.
- **같은 그룹의 인접한 표시 컬럼만 병합**된다. 숨겨진 컬럼은 스팬에서 제외되고,
  그로 인해 같은 그룹이 인접해지면 하나로 합쳐진다.
- `group` 없는 컬럼의 스팬은 `groupId: null` — 어댑터가 `rowspan=2`로 렌더링.
- `columnGroups`를 생략하면 그룹 ID 문자열이 그대로 헤더 텍스트가 된다.
- 어댑터는 상단 행(그룹 `colspan` + 단일 `rowspan`)과 하단 행(그룹 내 개별
  컬럼) 두 개의 `<tr>`로 렌더링한다.

## 상태 저장/복원 (State Persistence)

`getState()`가 직렬화 가능한 `GridPersistedState`를 반환하고,
`applyState()`로 복원한다 — localStorage에 JSON으로 저장하는 용도:

```ts
const state = grid.getState();
localStorage.setItem("grid-state", JSON.stringify(state));

grid.applyState(JSON.parse(localStorage.getItem("grid-state")!));
// 또는 부분 적용: grid.applyState({ sortState: saved.sortState })
```

포함 필드: `sortState`, `filters`, `searchText`, `pageIndex`/`pageSize`,
`groupBy`, `selectionMode`, `columnState`(order/width/visible/pinned).
없는 컬럼·필드를 참조하는 상태는 무시되므로 스키마 변경에도 안전하다.

## 컬럼 자동 너비 (Auto-Size)

```ts
grid.autoSizeColumn("name");                    // 단일 컬럼
grid.autoSizeAllColumns();                      // 전체 표시 컬럼
grid.autoSizeAllColumns((t) => ctx.measureText(t).width); // 정확한 측정
```

- 헤더 텍스트 + 표시 데이터의 `getCellText` 결과를 측정해 min/max 클램프된
  너비를 `setColumnWidth`로 적용한다.
- `measureText` 미지정 시 `text.length * 8 + 패딩`으로 추정하므로 DOM 없이도
  동작한다. 정확한 픽셀 측정이 필요하면 어댑터에서 canvas 기반 측정 함수를
  넘긴다.

## 셀 선택 (Selection Engine)

스냅샷에 다음 상태가 노출된다:

```ts
activeCell: { rowIndex, columnIndex } | null;   // 키보드 포커스 셀
selectedRange: { startRow, startCol, endRow, endCol } | null; // 정규화된 범위
selectionMode: 'single-cell' | 'multi-cell' | 'row';
```

인덱스는 `visibleData`/`visibleColumns` 기준이다 (정렬·필터·페이징·
재배치 결과에 대한 상대 위치). `GridOptions.selectionMode`로 초기 모드
지정, `setSelectionMode()`로 런타임 변경.

### 키보드 네비게이션 — `navigateCell(dir, extend)`

어댑터는 `KeyboardEvent.key`를 `CellNavigation`으로 매핑해 호출한다:

| 키 | `dir` | 동작 |
| -- | ----- | ---- |
| ArrowUp/Down/Left/Right | `up`/`down`/`left`/`right` | 인접 셀로 이동 (경계 클램프) |
| Home / End | `home`/`end` | 행의 첫/마지막 컬럼으로 이동 |
| PageUp / PageDown | `pageUp`/`pageDown` | 뷰포트 행 수 단위 이동 (`floor(viewportHeight/rowHeight)`, 비가상은 10) |
| Shift + 위 키 | `extend=true` | 앵커부터 범위 확장 (`single-cell` 모드 제외) |

동작 규칙:

- `single-cell` — 방향키는 활성 셀만 이동, `selectedRange`는 항상 null
- `multi-cell` — Shift 이동은 앵커→활성 셀의 사각 범위로 확장, 범위는 항상
  정규화(start ≤ end)되어 스냅샷에 노출
- `row` — 활성 행 전체 컬럼이 범위로 선택되고, Shift 이동은 행 단위 확장
- `Escape`는 코어가 아닌 어댑터가 처리 → `clearCellSelection()`

### scrollIntoView 연동

`navigateCell`/`setActiveCell`/`setCellRange` 후 활성 셀이 가상 스크롤
뷰포트 밖이면 코어가 `scrollTop`을 자동 보정하고 `virtual.scrollTop`을
스냅샷에 반영한다. 어댑터는 스냅샷 변경 시 `container.scrollTop`을
`virtual.scrollTop`과 동기화하면 된다 — 코어가 스크롤 위치의 진실 소스.
동기화로 발생하는 scroll 이벤트는 `handleScroll`의 동일값 가드로 무시된다.

### 선택 영역 집계 (상태바)

선택 범위(`selectedRange`, 없으면 `activeCell`)가 있으면 스냅샷
`selectionAggregates`에 스프레드시트 상태바식 집계가 들어간다:

```ts
{ cells: 6, count: 4, sum: 152, avg: 38, min: 12, max: 90 }
// cells: 범위 내 셀 수, count: 숫자 해석 가능 셀 수, 나머지는 숫자 없으면 null
```

선택이 없으면 `null`. 어댑터는 이를 하단 상태바(`mg-statusbar`)로 표시한다.

## 인라인 셀 편집 (Inline Cell Editing)

스냅샷에 다음 상태가 노출된다:

```ts
editingCell: { rowIndex, columnKey } | null; // visibleData 기준 행 + 컬럼 키
editValue: unknown;                           // 임시 입력 값
editError: string | null;                     // 유효성 검사 에러
```

### 편집 진입 트리거 (어댑터가 처리)

- 셀 **더블클릭** → `startEditing(r, c)`
- 활성 셀에서 **Enter** 또는 **F2** → `startEditing(r, c)` (기존 값으로 시작)
- 활성 셀에서 **인쇄 가능 문자 입력** → `startEditing(r, c, key)` (값 교체)

### 편집 종료

- **Escape** → `cancelEditing()` (저장 안 함)
- **Enter** → `commitEditing()` 성공 시 `navigateCell("down")`
- **Tab** → `commitEditing()` 성공 시 `navigateCell("right")`
  (Shift+Tab은 `"left"`), 검증 실패 시 편집 유지

### 커밋 처리

`commitEditing()`은 다음 순서로 동작한다:

1. `cellEditor === 'number'`이면 문자열 입력을 `Number`로 변환
2. `column.validate(value, row)` 검증 — `true`가 아니면 `editError`에
   메시지를 기록하고 편집 상태를 유지하며 `false` 반환 (저장 안 함)
3. `column.valueSetter`가 있으면 호출, 없으면 `row[field] = value`로
   rawData의 행 객체에 직접 기록 후 `notify()`로 파이프라인 재계산

### 커스텀 편집기

`cellEditor: 'custom'`이거나 어댑터의 에디터 확장점을 쓰는 경우
`getEditorContext(r, c)`가 `CellEditorContext`를 제공한다:
`{ row, column, value, editValue, error, setValue, commit, cancel }`.
각 어댑터는 이 컨텍스트를 React `renderEditor`, Vue `#editor-{field}` 슬롯,
Svelte `{#snippet editor}`에 전달한다.

## 편집 이력 (Undo/Redo)

`commitEditing()`으로 저장된 셀 변경과 `pasteTsv()` 붙여넣기가 이력 스택에
기록된다. 붙여넣기는 여러 셀이 바뀌어도 **1개 단위**로 기록된다:

```ts
grid.undo();    // 마지막 변경 되돌리기 → 성공 시 true
grid.redo();    // 다시 실행
grid.canUndo(); grid.canRedo();
```

- 되돌리기/다시 실행도 `valueSetter` 경로를 거친다 — 편집 커밋과 동일한
  쓰기 경로로 `oldValue`/`newValue`를 재적용한다.
- 어댑터는 `Ctrl/Cmd+Z` → `undo()`, `Ctrl/Cmd+Y`·`Ctrl/Cmd+Shift+Z` →
  `redo()`로 연결한다.
- 이력 크기는 `GridOptions.undoLimit`(기본값 100)로 제한하고,
  `undoLimit: 0`이면 이력 기록 자체를 끈다.
- `setData` 호출 시 이력은 초기화된다 (이전 행 객체 참조가 무의미해지므로).

## 고정 행과 총계 (Pinned Rows & Grand Totals)

### 상/하단 고정 행

`setPinnedTopRows(rows)` / `setPinnedBottomRows(rows)` 또는
`GridOptions.pinnedTopRows`/`pinnedBottomRows`로 필터·정렬과 무관하게
항상 표시되는 행을 추가한다 — 합계 행, 주석 행 등에 사용:

```ts
grid.setPinnedBottomRows([{ name: "합계", total: 1234 }]);
```

스냅샷 `pinnedTopRows`/`pinnedBottomRows`(`TData[]`)에 노출되고, 어댑터는
tbody 맨 위/맨 아래에 `getCellText`로 렌더링한다
(`mg-pinned-top-row`/`mg-pinned-bottom-row`).

### 전체 총계 (tfoot)

`aggregationFn`이 선언된 컬럼이 하나라도 있으면 스냅샷 `grandTotals`에
필터/검색을 통과한 전체 행(페이징 무관, 트리 자식 포함)에 대한 집계가
계산된다 — 서버 사이드 모드에서는 `null`:
`Record<field, aggregate>`. 어댑터는 `<tfoot>`의 `mg-total-row`로
표시하며 `formatAggregate()`로 포맷한다. 그룹 집계와 동일한 함수가 재사용된다.

## 행 드래그앤드롭 (Row Reordering)

`ColumnDef.rowDrag: true`인 컬럼의 셀에 드래그 핸들이 표시되고,
핸들을 드래그해 행 순서를 재정렬한다. 코어는 순서 이동과 드래그 상태를
관리하고, 어댑터가 HTML5 Drag and Drop 이벤트를 연결한다.

```ts
grid.moveRow(fromIndex, toIndex);        // visibleData 기준 인덱스 이동
grid.on("rowReorder", (e) => {
  // e = { fromIndex, toIndex, row } — 데이터 파이프라인 재계산 후 발행
});
```

드래그 라이프사이클 API (어댑터가 HTML5 DnD 이벤트에 연결):

| 메서드 | 동작 |
| ------ | ---- |
| `isRowDraggable()` | 드래그 가능 여부 — 정렬/그룹화/서버 사이드 모드에서는 `false` (표시 순서가 rawData와 무관하므로) |
| `beginRowDrag(rowIndex)` | 드래그 시작. 불가 상태/범위 밖이면 `false` |
| `updateRowDropPosition(gapIndex)` | 드롭 갭 갱신. `i`면 i번 행 위, `행 수`면 마지막 행 아래. 어댑터가 dragover의 포인터 Y로 계산 |
| `endRowDrag(commit)` | 종료. `true`면 갭을 최종 인덱스로 변환해 `moveRow` 실행, `false`면 취소 |
| `moveRow(from, to)` | 행 이동 + `rowReorder` 이벤트 발행. 성공 시 `true` |
| `getRowDragState()` | `{ draggingIndex, dropIndex }` 또는 `null` |

- 인덱스는 **visibleData 기준** — 필터가 걸려 있어도 화면에 보이는
  순서 관계가 유지되도록 rawData에서 목표 행 기준으로 재삽입한다.
- 스냅샷 `rowDrag`로 진행 상태를 노출 — 어댑터는 `draggingIndex`에
  `mg-row-dragging`(반투명), `dropIndex` 갭에 `mg-drop-before`/
  `mg-drop-after`(파란 가이드 라인)를 적용한다.
- `refresh()`(데이터 변경) 시 진행 중 드래그는 자동 해제된다.
- `Escape` 키는 어댑터에서 `endRowDrag(false)`로 연결된다.

## 데이터 파이프라인

```
rawData
  → applyFilters   (컬럼별 필터, AND)
  → applySearch    (전역 검색)
  → (filteredRowCount 확정)
  → applySort      (comparator 또는 collator)
  → applyPage      (pageIndex 클램프 + slice)
  → visibleData
```

필터/검색 변경 시 `pageIndex`는 0으로 리셋되고, 데이터가 줄어 현재
페이지가 범위를 벗어나면 마지막 페이지로 자동 보정된다.

## 가상 스크롤

대용량 데이터를 위한 뷰포트 계산 모듈 (`virtualScroll.ts`).
DOM에 의존하지 않는 순수 계산기 + GridCore 통합 API로 구성된다.

### computeVirtualScroll — 순수 계산기

```ts
import { computeVirtualScroll } from "@moda-grid/core";

const v = computeVirtualScroll({
  totalCount: 100_000,  // 전체 행 수
  rowHeight: 40,        // 행 높이 px
  viewportHeight: 600,  // 보이는 영역 높이 px
  scrollTop: 52_000,    // 현재 스크롤 위치 px
  overscan: 5,          // 상/하 여유분 행 수 (기본값 5)
});
// v = { startIndex, endIndex, startOffset, totalHeight }
// 렌더링: rows.slice(v.startIndex, v.endIndex)
// 상단 스페이서: v.startOffset px, 컨테이너: v.totalHeight px
```

| 출력 | 의미 |
| ---- | ---- |
| `startIndex` / `endIndex` | DOM에 그릴 인덱스 범위 (`endIndex` 미포함) |
| `startOffset` | 상단 스페이서 높이 (`startIndex * rowHeight`) |
| `totalHeight` | 전체 스크롤 높이 (`totalCount * rowHeight`) |

scrollTop은 `[0, totalHeight - viewportHeight]`로 클램프되고,
`totalCount === 0` 또는 `rowHeight <= 0`이면 빈 범위를 반환한다.

### GridCore 통합

```ts
const grid = new GridCore<User>({
  columns, data,
  virtualScroll: { rowHeight: 40, viewportHeight: 600 }, // 활성화
});

grid.subscribe(() => {
  const { virtual, virtualRows } = grid.getSnapshot();
  // virtualRows === visibleData.slice(virtual.startIndex, virtual.endIndex)
});
```

| API | 동작 |
| --- | ---- |
| `setVirtualScroll(config \| null)` | 런타임 활성/비활성 + 설정 변경 |
| `setViewportHeight(px)` | 컨테이너 리사이즈 대응 |
| `handleScroll(scrollTop)` | 스크롤 이벤트 핸들러 — **인덱스 범위가 바뀔 때만** 알림 |
| `getVirtualState()` | 현재 `VirtualScrollState` (비활성 시 null) |

스냅샷 추가 필드: `virtual`, `virtualRows`.
필터/정렬로 행 수가 바뀌면 `refresh()`에서 인덱스·`totalHeight`를
자동 재계산한다.

### 렌더링 패턴 (어댑터 공통)

```html
<div class="scroll-container" style="height:600px; overflow:auto"
     onscroll="grid.handleScroll(this.scrollTop)">
  <div style="height: {virtual.totalHeight}px; position: relative;">
    <div style="transform: translateY({virtual.startOffset}px)">
      <!-- virtualRows를 rowHeight 고정 높이로 렌더링 -->
    </div>
  </div>
</div>
```

`handleScroll`은 `startIndex`/`endIndex`가 변하지 않는 스크롤 이벤트를
건너뛰므로, 픽셀 단위 스크롤마다 리렌더되지 않는다.

## 테마 시스템 (CSS Variables)

`styles.css`의 모든 색상은 CSS 변수로 정의되어 있어, 변수만 재정의하면
그리드 전체 스타일을 제어할 수 있다.

```css
:root, .grid-theme-light {           /* 기본 라이트 팔레트 */
  --grid-bg-color: #ffffff;
  --grid-border-color: #e2e2e2;
  --grid-header-bg: #f7f7f8;
  --grid-text-color: #222222;
  --grid-row-hover-bg: #f4f6ff;
  --grid-row-selected-bg: #e5ecff;
  --grid-primary-color: #2563eb;
  /* …입력/패널/스켈레톤/고정 경계 등 파생 변수 — styles.css 참고 */
}

.grid-theme-dark { /* 동일 변수의 다크 팔레트 */ }
```

- **테마 전환** — 그리드 또는 상위 요소에 `.grid-theme-dark` 클래스를
  적용하면 된다(변수는 상속되므로 페이지 어디에 걸어도 된다). 클래스를
  지정하지 않으면 `:root`의 라이트 값이 기본으로 적용된다.
- **커스텀 테마** — 같은 이름의 변수만 덮어쓰면 된다:
  `.my-theme { --grid-primary-color: #9333ea; }`

### 커스텀 셀/행 클래스 (cellClass / rowClass)

조건부 스타일링은 클래스 주입으로 처리한다:

```ts
const columns = [{
  field: "age",
  // 셀 파라미터(value/row/rowIndex/column)로 조건부 클래스
  cellClass: ({ value }) => value >= 40 ? "cell-senior" : "",
}, {
  field: "role",
  // 행 파라미터(row/rowIndex) — 이 컬럼이 속한 모든 행에 적용
  rowClass: ({ row }) => row.role === "admin" ? "row-admin" : "",
}];

// 모든 행에 공통으로 적용하려면 GridOptions.rowClass
createGrid({ columns, data, rowClass: ({ rowIndex }) => rowIndex % 2 ? "row-odd" : "" });
```

- `rowIndex`는 `visibleData` 기준 — 필터/정렬 후 인덱스가 함수에 전달된다.
- `GridOptions.rowClass`와 모든 컬럼의 `rowClass`는 **합산**되어 행에 적용.
- 함수가 빈 문자열/`null`/`undefined`를 반환하면 클래스를 추가하지 않는다.
- 어댑터는 `grid.getRowClass(row, i)`/`grid.getCellClass(row, i, col)`로
  해석해 `<tr>`/`<td>`에 병합한다 (커스텀 어댑터도 동일하게 적용).

## 기능 추가 가이드

1. `types.ts` — 스냅샷/옵션 타입에 필드 추가
2. `grid.ts` — 내부 상태 필드 + `refresh()` 파이프라인 단계 추가 + 액션 메서드
3. `grid.test.ts` — 테스트 추가 (`pnpm test`)
4. 어댑터는 스냅샷에 포함되면 수정 없이 동작
