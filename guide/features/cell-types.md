# 셀 타입과 툴팁

텍스트 외 내장 셀 렌더링 — IBSheet의 이미지/버튼/링크/프로그레스 셀에
해당한다. 어댑터 전용 `renderCell`/슬롯 없이 선언만으로 렌더링된다.

## `ColumnDef.cellType`

```ts
const columns: ColumnDef<Product>[] = [
  // 이미지 — 셀 값을 src로 <img> 렌더링
  { field: "thumb", header: "이미지", cellType: "image",
    cellOptions: { alt: (row) => row.name } },

  // 버튼 — 셀 값을 라벨로 렌더링, 클릭 시 onClick
  { field: "action", header: "실행", cellType: "button",
    cellOptions: { onClick: (row) => runJob(row) } },

  // 링크 — 셀 값 또는 href() 결과로 <a> 렌더링 (기본 target=_blank)
  { field: "url", header: "링크", cellType: "link",
    cellOptions: { href: (row) => `/products/${row.id}`, target: "_self" } },

  // 프로그레스 — 숫자 값을 진행 바로 렌더링 (role=progressbar)
  { field: "progress", header: "진척", cellType: "progress",
    cellOptions: { max: 100 } },   // 기본값 100

  // HTML — 셀 값을 innerHTML로 렌더링 (아래 XSS 주의 참고)
  { field: "desc", header: "설명", cellType: "html" },
];
```

| cellType | 렌더링 | cellOptions |
| -------- | ------ | ----------- |
| `text` (기본) | 문자열 | — |
| `image` | `<img src={value}>` | `alt(row)` |
| `button` | `<button>` | `onClick(row)` |
| `link` | `<a href>` | `href(row)`, `target` |
| `progress` | 진행 바 + % 텍스트 | `max` (기본값 100) |
| `html` | `innerHTML` (raw HTML) | — |

> **⚠️ `html` 셀 타입은 XSS 주의**: 셀 값을 이스케이프 없이
> `innerHTML`로 주입한다. `<script>`/인라인 이벤트가 그대로 실행되므로
> **신뢰할 수 있는 데이터(서버에서 sanitize된 HTML 등)에만** 사용하고,
> 사용자 입력값을 담는 컬럼에는 쓰지 않는다. 단순 강조 정도면 `cellClass`
> 또는 어댑터 커스텀 셀이 안전한 대안이다.

- 버튼/링크 클릭은 행 선택 토글을 막지 않도록 `stopPropagation`된다.
- 프로그레스는 `aria-valuenow`/`aria-valuemax`를 포함한다.
- 편집(`cellEditor`)과 병용 가능 — 편집 중에는 에디터가 렌더링된다.
- 더 복잡한 렌더링은 어댑터 커스텀 셀(React `renderCell`, Vue
  `#cell-{field}` 슬롯, Svelte `cell` snippet)을 사용한다.

## 셀 툴팁 (`ColumnDef.tooltip`)

```ts
{ field: "name", tooltip: "전체 이름" },                    // 고정 문자열
{ field: "memo", tooltip: ({ value }) => value == null ? null : String(value) },
// 함수는 { value, row, column }을 받고 문자열 또는 null을 반환
```

해석된 문자열은 셀 `title` 속성으로 렌더링된다 (네이티브 툴팁).
null/undefined 반환 시 툴팁 없음. **셀 유효성 에러가 있으면 에러 메시지가
툴팁보다 우선**한다.
