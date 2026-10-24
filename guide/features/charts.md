# 차트 브릿지 (`@moda-grid/charts`)

`@moda-chart/core`를 그리드에 연결하는 선택적 브릿지 패키지 — 스파크라인
셀, 선택 영역 차트, 상세 행 차트, 테마 동기화를 제공한다. 차트 렌더가
DOM/SVG 기반이라 **어댑터 없이 5개 렌더러(React·Vue 3·Vue 2.7·Svelte·
vanilla) 모두에서 동일하게 동작**한다.

`@moda-grid/core`는 차트에 의존하지 않는다 — 차트를 쓰지 않는 사용자에게
차트 코드가 포함되지 않도록 브릿지가 별도 패키지로 분리돼 있다
(`docs-internal/chart-integration.md` 설계).

```bash
pnpm add @moda-grid/charts @moda-chart/core
```

```ts
import "@moda-grid/charts/styles.css"; // 패널 스타일
import { sparklineColumn, chartMenuItem } from "@moda-grid/charts";
```

`@moda-chart/core`는 peer dependency — 프로젝트가 버전을 직접 고른다.

## 스파크라인 컬럼 (`sparklineColumn`)

숫자 배열 필드를 인라인 스파크라인 SVG로 렌더하는 `ColumnDef`를 만든다.
`cellType: "html"` 경로를 쓰므로 어떤 렌더러에서도 추가 코드가 필요 없다.

```ts
const columns: ColumnDef<User>[] = [
  ...userColumns,
  {
    // row.trend: number[] 필드를 읽는다
    ...sparklineColumn<User>("trend", { type: "area", header: "추이" }),
    width: 130,
  },
];
```

| 옵션     | 기본값       | 내용                                                     |
| -------- | ------------ | -------------------------------------------------------- |
| `type`   | `"line"`     | 스파크라인 타입 — `line`/`area`/`bar`/`column`/`winloss` |
| `header` | `field`      | 헤더 라벨                                                |
| `values` | `row[field]` | 행 → 숫자 배열 추출기. 비숫자/무한대는 null로 정규화된다 |
| 그 외    | —            | `SparklineOptions`(색상·폭 등)가 그대로 통과             |

반환 컬럼의 `formatter`가 SVG 마크업을 만든다 — `editable`/`formatter`를
덮어쓰면 스파크라인이 깨진다. 셀 값 자체는 배열로 유지되므로 정렬/복사는
배열 기준으로 동작한다.

## 선택 영역 차트 (`selectionToChart` + `chartMenuItem`)

`getSelectedCells()` 결과를 `{ series, categories }`로 변환한다 — 숫자 셀만
시리즈가 되고, `categoryField` 컬럼 값이 카테고리 축이 된다. 시리즈 순서는
`seriesFields`로 제한/정렬할 수 있다.

```ts
import { selectionToChart, chartMenuItem } from "@moda-grid/charts";

// 직접 변환
const input = selectionToChart(grid.getSelectedCells(), {
  type: "bar",
  categoryField: "name",
});
mountChart(el, { type: input.type ?? "line", ...input });

// 컨텍스트 메뉴 항목 — 선택에 숫자 셀이 없으면 비활성으로 렌더된다
<DataGrid contextMenu={(ctx) => [
  chartMenuItem(grid, { categoryField: "name", type: "bar" }),
  ...defaultContextMenuItems(ctx),
]} />
```

메뉴를 클릭하면 `.mg-chart-panel` 플로팅 패널이 열린다 — 헤더 드래그로
이동, `Esc`/닫기 버튼으로 닫기, 닫힐 때 차트 인스턴스도 함께 파괴된다.
패널은 클릭 시점 선택의 스냅샷이다 — 이후 선택 변경을 반영하려면
`panel.update`(또는 `onOpen`으로 받은 핸들)를 직접 호출한다.

`chartMenuItem` 옵션:

| 옵션            | 내용                                                            |
| --------------- | --------------------------------------------------------------- |
| `type`          | 차트 타입(기본 `"line"`)                                        |
| `categoryField` | 카테고리 축으로 쓸 컬럼 field                                   |
| `seriesFields`  | 포함할 숫자 컬럼 화이트리스트(순서=시리즈 순서)                 |
| `title`         | 패널/차트 제목                                                  |
| `id`·`label`    | 메뉴 항목 커스터마이즈(기본 `chart`, `"선택 영역 차트"`)        |
| `panel`         | `openChartPanel` 옵션(`x`/`y`/`container`/`onClose`) 오버라이드 |
| `onChart`       | 변환 결과 훅 — 패널 대신 직접 마운트하고 싶을 때                |
| `onOpen`        | 패널이 열린 뒤 호출 — 테마 동기화 등 후처리                     |

## 플로팅 패널 (`openChartPanel`)

메뉴 없이 패널만 직접 열 수도 있다 — 반환 핸들의 `update`는 차트
`updateDelta`를 감싼다.

```ts
const panel = openChartPanel({
  type: "bar",
  series: [{ name: "매출", values: [10, 20, 30] }],
  categories: ["1월", "2월", "3월"],
  x: 120,
  y: 60,
});
panel.update({ title: "Q1" });
panel.close(); // 차트 destroy + DOM 제거
```

## 상세 행 차트 (`chartDetailRenderer`)

vanilla `detailRenderer`에 차트를 싣는 래퍼 — 노드가 DOM에서 떨어지면
차트를 자동 파괴한다(MutationObserver 기반).

```ts
detailRenderer: chartDetailRenderer<User>((row, host) => ({
  type: "line",
  series: [{ name: row.name, values: row.trend ?? [] }],
}));
```

반환 함수는 `Node`를 반환하는 vanilla `detailRenderer` 시그니처 전용이다
— 프레임워크 어댑터의 상세 패널은 프레임워크 노드를 반환하므로 거기선
ref/슬롯에서 `mountChart`를 직접 쓴다. 그리드 파괴 시 남은 인스턴스는
`destroyDetachedCharts()`로 일괄 정리할 수 있다.

## 테마 동기화 (`gridThemeToChartTheme` / `syncChartTheme`)

그리드 테마를 차트 테마로 번역한다 — `GridTheme`(`"light"`/`"dark"`/`vars`
오브젝트)의 `--grid-*` 변수를 `--chart-*` 대응 변수로 매핑한다.

```ts
// 마운트 옵션에 직접
mountChart(el, { ...input, theme: gridThemeToChartTheme(theme) });

// 런타임 동기화 — 그리드 테마 변경 이벤트에 연결
syncChartTheme(mounted, newTheme); // mounted.chart.updateDelta({theme})
```

데모 앱들은 `onOpen` 콜백에서 `syncChartTheme(panel, theme)`을 호출해
그리드 테마를 패널 차트에 반영한다.

## DOM 계약

- 패널 — `.mg-chart-panel`(`.mg-chart-panel-title`/`-body`/`-close`)
- 스파크라인/차트 — moda-chart의 `.mc-*` 클래스(`.mc-sparkline`,
  `.mc-root[data-theme]` 등)

두 접두어를 섞지 않는다 — 그리드 측 요소만 `mg-`, 차트 측은 `mc-`.
