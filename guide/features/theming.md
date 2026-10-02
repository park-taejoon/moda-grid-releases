# 테마 / 커스텀 스타일 (Theming)

모든 색상과 간격·타이포는 `--grid-*` CSS 변수 기반. `theme` prop,
`density` prop, 변수 재정의, 클래스 주입으로 그리드 전체 스타일을 제어한다.

## `theme` 옵션 — 세 가지 형태

`theme` prop/`mountGrid` 옵션은 문자열 프리셋과 오브젝트 오버라이드를
모두 받는다:

```tsx
// 1. 내장 팔레트 — "light" | "dark"
<DataGrid columns={cols} data={rows} theme="dark" />;
mountGrid(el, { columns, data, theme: "dark" });

// 2. 오브젝트 — 베이스 팔레트 위에 --grid-* 변수만 덮어쓴다
<DataGrid
  columns={cols}
  data={rows}
  theme={{
    base: "light",
    vars: {
      primaryColor: "#0d9488", // camelCase → --grid-primary-color
      "--grid-font-size": "13px", // --grid-* 이름도 그대로 가능
      borderRadius: "8px",
      cellPaddingY: "4px",
    },
  }}
/>;
```

- `"dark"` → 루트에 `.grid-theme-dark`, `"light"` → `.grid-theme-light` 적용
  (다크 페이지 안의 라이트 그리드도 가능).
- `{ base, vars }` — `base`는 팔레트 클래스(생략 시 기본 라이트 팔레트
  `:root` 위에 얹는다), `vars`는 루트에 인라인 `--grid-*` 변수로 적용된다.
  키는 camelCase(`primaryColor`)와 변수명(`--grid-primary-color`) 모두 허용.
- CSS 파일 없이 JS만으로 테마를 만들 수 있고, prop/옵션을 바꾸면
  런타임에 전환된다 — 이전 적용분(클래스·인라인 변수)은 자동으로 제거된다.
- 생략하면 아무것도 추가하지 않는다 — 상위 요소/`:root` 팔레트를 상속.

### 내장 프리셋 — `gridThemePresets`

자주 쓰는 조합을 프리셋으로 제공한다 — `theme`에 그대로 전달하거나
`vars`를 펼쳐 커스텀 팔레트의 출발점으로 쓴다:

```ts
import { gridThemePresets } from "@moda-grid/react"; // 모든 패키지 동일

// 바이올렛 포인트 — primary/헤더/선택 배경을 보라 계열로
<DataGrid theme={gridThemePresets.violet} ... />;

// 고대비 — 저시력·프로젝터 환경 (굵은 경계선 + 진한 선택 배경)
<DataGrid theme={gridThemePresets.highContrast} ... />;

// 프리셋 위에 추가 오버라이드
mountGrid(el, {
  columns,
  data,
  theme: {
    base: "dark",
    vars: { ...gridThemePresets.violet.vars, fontSize: "15px" },
  },
});
```

| 프리셋         | base    | 특징                                                     |
| -------------- | ------- | -------------------------------------------------------- |
| `violet`       | `light` | 보라 계열 포인트 색 + 둥근 모서리 (브랜드 테마 예시)     |
| `highContrast` | `light` | 검정 텍스트·진한 경계·파란 선택 배경 — 접근성/프로젝터용 |

## `density` 옵션 — 밀도 프리셋

테마와 독립적으로 폰트 크기·셀 패딩을 일괄 조정한다:

```tsx
<DataGrid columns={cols} data={rows} density="compact" />;
mountGrid(el, { columns, data, density: "comfortable" });
```

| 값            | 클래스                    | 폰트 | 셀 패딩     |
| ------------- | ------------------------- | ---- | ----------- |
| `standard`    | (없음 — 기본값)           | 14px | 8px / 12px  |
| `compact`     | `.mg-density-compact`     | 12px | 3px / 8px   |
| `comfortable` | `.mg-density-comfortable` | 15px | 12px / 16px |

밀도 클래스는 `--grid-font-size`/`--grid-cell-padding-*` 변수만
오버라이드한다 — 커스텀 값이 있으면 `theme.vars`로 함께 지정하는 게
예측 가능하다(테마 인라인 변수가 클래스보다 우선한다).

## 클래스 직접 적용

그리드 또는 상위 요소에 `.grid-theme-dark` 클래스를 적용한다 (CSS 변수는
상속되므로 페이지 어디에 걸어도 된다):

```html
<body class="grid-theme-dark">
  <!-- 페이지 전체 다크 -->
  <div class="grid-theme-dark"><!-- 이 그리드만 다크 --></div>
</body>
```

클래스를 지정하지 않으면 `:root`의 라이트 팔레트가 기본 적용된다.

## 행 줄무늬 — `striped`

zebra 줄무늬(짝수 표시 행 배경)를 켠다 — 타사 그리드 `Alternate` 스타일 대응:

```tsx
<DataGrid columns={cols} data={rows} striped />; // 어댑터 prop
mountGrid(el, { columns, data, striped: true }); // mountGrid 옵션
grid.setStriped(true); // 런타임 토글
```

- `tbody tr:nth-child(even)`에 `--grid-stripe-bg`를 적용한다 — 정렬/필터/
  그룹으로 표시 순서가 바뀌어도 화면 기준 짝수 행에 일관 적용.
- hover·선택·I/U/D 상태·고정 행·find 하이라이트는 줄무늬보다 우선한다.
- 고정(pinned) 셀에는 **불투명** `--grid-stripe-pinned-bg`를 사용한다 —
  반투명을 쓰면 sticky 아래로 스크롤 콘텐츠가 비치는 잔상이 생긴다.
- 색상은 `--grid-stripe-bg` / `--grid-stripe-pinned-bg` 변수로 재정의 가능
  (다크 테마 값 내장).

### 네이티브 컨트롤 — `color-scheme`

테마 클래스는 `color-scheme: light|dark`를 함께 지정한다 — 그리드 내부의
네이티브 스크롤바·checkbox·select 화살표 등도 테마에 맞춰 렌더링된다.

## CSS 변수

```css
:root,
.grid-theme-light {
  /* 색상 */
  --grid-bg-color: #ffffff;
  --grid-border-color: #e2e2e2;
  --grid-header-bg: #f7f7f8;
  --grid-text-color: #222222;
  --grid-row-hover-bg: #f4f6ff;
  --grid-row-selected-bg: #e5ecff;
  --grid-primary-color: #2563eb;
  --grid-on-accent-color: #ffffff; /* primary·액센트 표면 위 텍스트 */

  /* 간격·타이포 — density 프리셋과 theme.vars가 오버라이드 */
  --grid-font-size: 14px;
  --grid-font-size-sm: 12px; /* 메뉴·배지·오버레이 등 보조 텍스트 */
  --grid-font-size-xs: 11px;
  --grid-cell-padding-y: 8px;
  --grid-cell-padding-x: 12px;
  --grid-border-radius: 4px; /* 버튼·인풋·칩 등 작은 컨트롤 */
  --grid-panel-radius: 8px; /* 패널·메뉴·토스트 등 큰 표면 */
  /* 행 상태 색도 변수: --grid-status-i/u/d-color, --grid-status-i/u/d-bg */
  /* 입력/패널/스켈레톤/고정 경계 등 파생 변수 — styles.css 참고 */
}
```

커스텀 테마는 같은 이름의 변수만 덮어쓴다:

```css
.my-theme {
  --grid-primary-color: #9333ea;
  --grid-row-selected-bg: #f3e8ff;
}
```

## 조건부 스타일 — cellClass / rowClass

```ts
const columns = [
  {
    field: "age",
    // 셀 파라미터 { value, row, rowIndex, column } → 조건부 클래스
    cellClass: ({ value }) => (Number(value) >= 40 ? "cell-senior" : ""),
  },
  {
    field: "role",
    // 행 파라미터 { row, rowIndex } — 이 컬럼이 속한 모든 행에 적용
    rowClass: ({ row }) => (row.role === "admin" ? "row-admin" : ""),
  },
];

// 모든 행에 공통 적용 — GridOptions.rowClass
createGrid({
  columns,
  data,
  rowClass: ({ rowIndex }) => (rowIndex % 2 ? "row-odd" : ""),
});
```

```css
.cell-senior {
  font-weight: 600;
  color: #b45309;
}
.row-admin {
  background: #fff7ed;
}
.row-odd {
  background: #fafafa;
}
```

- `rowIndex`는 `visibleData` 기준 — 필터/정렬 후 인덱스가 함수에 전달된다.
- `GridOptions.rowClass`와 모든 컬럼의 `rowClass`는 **합산**되어 `<tr>`에 적용.
- 함수가 빈 문자열/`null`/`undefined`를 반환하면 클래스를 추가하지 않는다.
- 클래스 값은 문자열 상수도 가능: `cellClass: "text-right"`.
- 커스텀 렌더러(비-어댑터)는 `grid.getRowClass(row, i)` /
  `grid.getCellClass(row, i, col)`로 해석 결과를 얻는다.

## 인라인 스타일 — cellStyle

클래스 없이 셀 하나만 직접 색을 바꿀 때 (타사 그리드 조건부 색상 대응).
CSS 문자열, kebab-case 속성 맵, 또는 셀 파라미터를 받는 함수를 지정한다:

```ts
{
  field: "age",
  cellStyle: ({ value }) =>
    Number(value) >= 40 ? { "background-color": "#fee2e2", color: "#b91c1c" } : null,
}
// 또는 CSS 문자열: cellStyle: "background-color: #fee2e2; color: #b91c1c"
```

- 반환값은 `Record<string, string>`(kebab-case) 또는 CSS 문자열 —
  코어가 문자열을 맵으로 파싱해 어댑터가 `<td style>`에 병합한다.
- 커스텀 렌더러는 `grid.getCellStyle(row, rowIndex, col)`로 맵을 얻는다.

## 인라인 스타일 — rowStyle

`rowClass`의 인라인 스타일 버전 — 행(`<tr>`) 전체에 스타일을 직접 적용한다.
CSS 문자열/kebab-case 맵/행 파라미터 함수 모두 지원:

```ts
{
  field: "role",
  rowStyle: ({ row }) =>
    row.role === "admin" ? { "background-color": "rgba(99,102,241,.15)" } : null,
}

// 모든 행 공통 — GridOptions.rowStyle
createGrid({
  columns, data,
  rowStyle: "border-top: 2px solid #e5e7eb",
});
```

- `GridOptions.rowStyle`과 모든 컬럼의 `rowStyle`이 **병합**되어 `<tr>`에 적용 —
  같은 속성이면 나중(컬럼) 쪽이 우선한다.
- 커스텀 렌더러는 `grid.getRowStyle(row, rowIndex)`로 맵을 얻는다.
- `<tr>`에 적용되므로 `background-color`는 셀 배경이 투명할 때만 보인다.

## 주요 구조 클래스 (커스텀 스타일링 대상)

| 클래스                                                       | 대상                                                     |
| ------------------------------------------------------------ | -------------------------------------------------------- |
| `.mg-table`                                                  | 그리드 테이블                                            |
| `.mg-group-row`                                              | 그룹 헤더 행                                             |
| `.mg-cell-active` / `.mg-cell-selected`                      | 활성 셀 / 선택 범위                                      |
| `.mg-cell-editing` / `.mg-edit-error`                        | 편집 중 셀 / 에러                                        |
| `.mg-pinned` / `.mg-pin-left-edge` / `.mg-pin-right-edge`    | 고정 컬럼                                                |
| `.mg-pinned-top-row` / `.mg-pinned-bottom-row`               | 고정 행                                                  |
| `.mg-total-row`                                              | 총계 `<tfoot>` 행                                        |
| `.mg-skeleton-row` / `.mg-loading`                           | 서버 모드 스켈레톤/로딩                                  |
| `.mg-loading-host` / `.mg-loading-overlay`                   | 로딩 오버레이 활성 루트 / 오버레이 (`loading` 옵션·prop) |
| `.mg-statusbar` / `.mg-colctl-panel`                         | 상태바 / 컬럼 관리 팝오버                                |
| `.mg-row-drag-handle` / `.mg-drop-before` / `.mg-drop-after` | 행 드래그                                                |
| `.mg-empty`                                                  | 빈 그리드 행                                             |
| `.mg-striped`                                                | 줄무늬 활성 루트 (`striped` 옵션)                        |
| `.mg-density-compact` / `.mg-density-comfortable`            | 밀도 프리셋 활성 루트 (`density` 옵션)                   |
