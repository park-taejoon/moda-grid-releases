# 순수 JavaScript 가이드 — CDN 번들

프레임워크 없이 `<script>` 태그로 사용하는 헤드리스 코어 번들.
**DOM 렌더링은 직접 작성한다** — 번들에는 `GridCore`와 헬퍼만 들어 있다.

## 설치 (CDN)

```html
<link rel="stylesheet" href="https://grid.modaolive.com/style.css" />
<script src="https://grid.modaolive.com/moda-grid.js"></script>
```

`window.ModaGrid`에 코어의 모든 export가 노출된다: `GridCore`, `createGrid`,
`defineColumns`, `computeVirtualScroll`, `matchFilterValue`, `buildCsv`,
`downloadCsv`, `formatAggregate`, `DEFAULT_COLUMN_WIDTH` 등.

## 기본 사용

```html
<input id="search" placeholder="검색…" />
<div id="app"></div>

<script>
  const { GridCore } = window.ModaGrid;

  const columns = [
    { field: "id",   header: "ID",  width: 60 },
    { field: "name", header: "이름", filterable: true },
    { field: "role", header: "역할" },
    { field: "age",  header: "나이", filterType: "number" },
  ];
  const users = [
    { id: 1, name: "Hana",  role: "admin",  age: 42 },
    { id: 2, name: "Daeho", role: "editor", age: 36 },
  ];

  const grid = new GridCore({ columns, data: users });

  function render() {
    const snap = grid.getSnapshot();
    document.getElementById("app").innerHTML = `
      <table class="mg-table">
        <thead><tr>
          ${snap.visibleColumns
            .map((c) => `<th data-field="${c.field}">${c.header ?? c.field}</th>`)
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

  grid.subscribe(render);          // 상태 변경 → 리렌더

  // 헤더 클릭 → 정렬 (Shift+클릭 = 다중 정렬)
  document.getElementById("app").addEventListener("click", (e) => {
    const th = e.target.closest("th[data-field]");
    if (th) grid.toggleSort(th.dataset.field, e.shiftKey);
  });
  document.getElementById("search").addEventListener("input", (e) => {
    grid.setSearch(e.target.value);
  });

  render();
</script>
```

## 코어 API를 직접 쓸 때의 규칙

어댑터가 해주는 일을 직접 구현한다고 생각하면 된다:

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
grid.getCellText(row, col);          // formatter/valueGetter 적용 문자열
grid.getCellClass(row, rowIndex, c); // cellClass 해석 결과
grid.getRowClass(row, rowIndex);     // rowClass 해석 결과
```

## 가상 스크롤 (순수 JS)

```js
const grid = new GridCore({
  columns, data,
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
import { GridCore } from "@moda-grid/core";
import "@moda-grid/core/styles.css";
```
