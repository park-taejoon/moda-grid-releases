# 순수 JavaScript 가이드 — CDN 번들

프레임워크 없이 `<script>` 태그로 사용하는 두 가지 방식:

- **`mountGrid`** — 완성된 DOM 그리드를 한 번에 마운트 (타사 그리드 `create` 방식)
- **`GridCore`** — 헤드리스 코어만 사용, DOM 렌더링은 직접 작성

## 설치 (CDN)

```html
<link rel="stylesheet" href="https://grid.modaolive.com/style.css" />
<script src="https://grid.modaolive.com/moda-grid.js"></script>
```

`window.ModaGrid`에 코어의 모든 export가 노출된다: `mountGrid`, `GridCore`,
`createGrid`, `defineColumns`, `formatAggregate` 등.

## mountGrid — 완성된 그리드 (권장)

```html
<div id="grid" style="height: 480px"></div>

<script>
  const { mountGrid } = window.ModaGrid;

  const mounted = mountGrid(document.getElementById("grid"), {
    columns: [
      { field: "id", header: "ID", width: 60 },
      { field: "name", header: "이름", filterable: true },
      { field: "role", header: "역할", filterable: true, filterType: "set" },
      { field: "age", header: "나이", filterable: true },
      { field: "done", header: "완료", cellEditor: "checkbox" },
    ],
    data: [
      { id: 1, name: "Hana", role: "admin", age: 42, done: true },
      { id: 2, name: "Daeho", role: "editor", age: 36, done: false },
    ],
    height: 460, // 가상 스크롤 (rowHeight와 함께)
    rowHeight: 36,
    rowNumbers: true, // 행 번호 컬럼
    rowCheckboxes: true, // 행 체크박스 + 헤더 전체선택
    rowStatus: true, // I/U/D 상태 컬럼
    searchBox: true, // 상단 검색 입력 (데이터 필터)
    findBox: true, // 찾기 바 — 셀 텍스트 검색 후 매치로 이동
    pager: true, // 하단 페이저/행수
    pageSize: 20,
    // 행 상세 패널 — 첫 셀에 ▸ 토글, 펼친 행 아래 전체 너비 패널
    detailRenderer: (row) => `${row.name} 상세 내용`, // Node도 가능
    // 선언형 이벤트 (grid.on과 동일)
    events: {
      afterEdit: (e) => console.log(e.row.id, e.column.field, e.newValue),
    },
  });

  // 인스턴스 API는 mounted.grid로 접근
  mounted.grid.exportToCsv({ filename: "users.csv" });

  // 정리
  mounted.destroy();
</script>
```

### mountGrid가 렌더링하는 것

- 헤더 클릭 정렬 (Shift = 다중 정렬), `filterable` 컬럼의 필터 행
- 다단계 헤더 (`columnGroups` + `ColumnDef.group` → `.mg-colgroup` 2행 헤더)
- 헤더 리사이즈 핸들 (`resizable`, 기본 true) + 헤더 드래그 컬럼 재배치
  (`reorderable`, 기본 true)
- `groupPanel`: 그룹된 컬럼 칩 표시 + 클릭으로 해제
- `autoRowHeight`: `multiLine` 셀 내용만큼 행 높이 자동 확장
- 더블클릭 인라인 편집 (text/number/select/date/checkbox/multiselect/radio/textarea)
- 셀 타입: `image` / `button` / `link` / `progress` / `html`
- 키보드: 방향키/Home/End/PageUp·Down 이동, Enter/F2 편집 진입,
  Delete/Backspace 내용 지우기(`clearRange`), Ctrl+C/V 복사·붙여넣기,
  Ctrl+Z/Y·Ctrl+Shift+Z 실행취소/재실행, Ctrl+A 전체 선택(`selectAll`),
  Ctrl+D/R 아래/오른쪽 채우기(`fillDown`/`fillRight`), Tab 셀 이동,
  Ctrl+X 잘라내기(`cutSelectionTsv`), Ctrl+Home/End·Ctrl+방향키 점프,
  Shift+Space 행 선택 / Ctrl+Space 열 선택, Ctrl+Enter 범위 입력
  (`fillActiveToSelection`), `fillSeries` 시리즈 채우기 옵션,
  키보드 이동 시 수평 스크롤 자동 추적 (`scrollRequest.columnIndex`)
- 행 선택, `rowStatus` I/U/D 마킹, `rowNumbers`
- 그룹/소계/트리 행, 셀 병합(rowSpan/colSpan), pinned 컬럼
- `aggregationFn` 컬럼의 합계 푸터(`tfoot`)
- 셀/헤더 컨텍스트 메뉴 (`contextMenu`/`headerContextMenu` 옵션)
- `height`+`rowHeight` 가상 스크롤, `appendScroll` 자동 로드
- `pager`: `pageSize > 0`이면 이전/다음 버튼 + 페이지 번호 버튼 + 페이지 정보
- 마우스 드래그 셀 범위 선택 (`selectionMode: "multi-cell"`/`"row"`)
- `col.rowDrag` 행 드래그 재정렬 — 핸들 드래그 + 드롭 가이드 라인
  (`beginRowDrag`/`updateRowDropPosition`/`endRowDrag` → `moveRow`)
- `findBox`: 셀 찾기 바 — Enter 다음/Shift+Enter 이전 매치 이동 (어댑터 `findBox` prop과 동일)
  (`findCells`/`findNext` 기반, `searchBox`의 데이터 필터와는 별개)
- `columnController`: 우상단 컬럼 관리 도구 — 표시/숨김 체크박스 팝오버
  - 전체 선택/해제
- `filterToggle`: 필터 행 표시/숨김 토글 버튼 (filterable 컬럼이 있을 때만)
- `headerFilters`: filterable 컬럼 헤더에 ▾ 값 체크리스트 드롭다운
  (엑셀 autofilter — 어댑터 `headerFilters` prop과 동일)
- `pivotPanel`: 상단에 필드/행/열/값 존 패널 — 칩 드래그로 피벗 구성
  (엑셀 피벗 필드 목록 — 어댑터 `pivotPanel` prop과 동일)
- `theme`: 그리드 자체 테마 (`"light" | "dark"`) — 루트에 `grid-theme-*`
  클래스 적용, 생략 시 상위 팔레트 상속 (어댑터 `theme` prop과 동일)
- `statusBar`: 선택 영역 집계 상태바 (셀 수/개수/합계/평균/최소/최대,
  기본 true)
- `pageSizeOptions`: 페이저에 페이지 크기 셀렉트 (예: `[10, 20, 50]`)
- `rowResizable`: 행 번호 셀 하단 드래그로 행 높이 조정 (`rowNumbers` 필요)

`GridOptions`의 모든 옵션(`treeData`, `pivot`, `groupSubtotals`, `locale`,
`pinnedTopRows` …)을 그대로 받는다.

## GridCore 직접 렌더링 (고급)

완전한 렌더링 제어가 필요하면 코어만 사용한다:

```html
<div id="app"></div>
<script>
  const { GridCore } = window.ModaGrid;
  const grid = new GridCore({ columns, data });

  function render() {
    const snap = grid.getSnapshot();
    document.getElementById("app").innerHTML = `
      <table class="mg-table">
        <thead><tr>
          ${snap.visibleColumns
            .map(
              (c) => `<th data-field="${c.field}">${c.header ?? c.field}</th>`,
            )
            .join("")}
        </tr></thead>
        <tbody>
          ${snap.rows
            .map(
              (row) =>
                `<tr>${snap.visibleColumns
                  .map((c) => `<td>${grid.getCellText(row, c)}</td>`)
                  .join("")}</tr>`,
            )
            .join("")}
        </tbody>
      </table>`;
  }

  grid.subscribe(render);
  document.getElementById("app").addEventListener("click", (e) => {
    const th = e.target.closest("th[data-field]");
    if (th) grid.toggleSort(th.dataset.field, e.shiftKey);
  });
  render();
</script>
```

### 코어 API를 직접 쓸 때의 규칙

1. `grid.subscribe(render)`로 리렌더 연결 — 모든 상태 변경은
   `grid.*()` 메서드가 `notify()`를 부르므로 수동 갱신 불필요.
2. `snapshot.visibleColumns` / `snapshot.rows` 를 그대로 렌더링 —
   컬럼 순서·고정 파티션·필터/정렬/페이징이 이미 반영되어 있다.
3. 편집/선택/가상 스크롤 같은 고급 기능도 코어 상태(`snapshot.editingCell`,
   `snapshot.virtualRows` …)를 읽어 렌더링하고 이벤트는 코어 메서드로
   전달하면 된다. 각 기능의 렌더링 계약은 [features/](../features/) 문서의
   "어댑터 렌더링" 절을 참고.

## 표시 텍스트/클래스 헬퍼

```js
grid.getCellText(row, col); // formatter/valueGetter 적용 문자열
grid.getCellClass(row, rowIndex, c); // cellClass 해석 결과
grid.getRowClass(row, rowIndex); // rowClass 해석 결과
```

## 가상 스크롤 (순수 JS, 수동 렌더링)

```js
const grid = new GridCore({
  columns,
  data,
  virtualScroll: { rowHeight: 40, viewportHeight: 600 },
});

container.onscroll = () => grid.handleScroll(container.scrollTop);
// render()에서 snap.virtualRows만 그리고
// 상단 스페이서 = snap.virtual.startOffset, 컨테이너 높이 = snap.virtual.totalHeight
```

상세 패턴: [features/virtual-scroll.md](../features/virtual-scroll.md).

## npm으로 코어만 사용

```bash
npm install @moda-grid/core
```

```ts
import { GridCore, mountGrid } from "@moda-grid/core";
import "@moda-grid/core/styles.css";
```
