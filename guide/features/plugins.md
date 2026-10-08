# 기능 등록과 플러그인 (Features & Plugins)

`GridOptions.features`로 활성화할 기능을 선택하고, 커스텀 기능을
플러그인처럼 등록할 수 있다. 슬림 엔트리(`@moda-grid/core/slim`)와 함께
쓰면 미사용 기능의 코드가 번들에서 완전히 제거된다.

## 기본 동작

```ts
import { GridCore } from "@moda-grid/core";

const grid = new GridCore({ columns, data }); // features 미지정 — 전체 기능 활성
```

`features`를 넘기면 그 목록만 활성화된다:

```ts
import {
  GridCore,
  columnsFeature,
  sortingFeature,
  pagingFeature,
} from "@moda-grid/core";

const grid = new GridCore({
  columns,
  data,
  features: [columnsFeature(), sortingFeature(), pagingFeature()],
});
```

- 미등록 기능의 공개 API(`setFilter` 등)는 no-op이고 개발 모드에서
  경고를 출력한다.
- 미등록 기능과 연관된 옵션(`pageSize` 등)을 넘기면 생성 시 경고가 난다.
- 스냅샷 필드(`selectedRowIds`, `virtual`, `pageIndex` …)는 미등록이어도
  빈 값으로 항상 존재한다 — 어댑터는 그대로 동작한다.
- `columns`를 빼면 컬럼은 `GridOptions.columns`가 그대로 통과한다
  (정렬·숨김·재배치 같은 상태만 꺼진다).

## 기능 팩토리 목록

`@moda-grid/core`(전체 엔트리)와 `@moda-grid/core/slim` 모두에서 export된다.
`name` 열은 등록표 키 — `internals.ctl(name)` 조회와 덮어쓰기 매칭에
쓰는 식별자다. "연관 옵션"은 해당 기능을 `features`에서 뺐을 때 지정하면
개발 경고가 나는 `GridOptions` 키다.

| name             | 팩토리                  | 담당 범위                                                  | 연관 옵션                                                                         |
| ---------------- | ----------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `columns`        | `columnsFeature`        | 컬럼 상태(너비/순서/핀/숨김)·그룹 헤더·자동 너비           | `defaultColDef`, `columnGroups`                                                   |
| `sorting`        | `sortingFeature`        | 정렬 모델·다중 정렬·정렬 후처리                            | `postSort`                                                                        |
| `filtering`      | `filteringFeature`      | 컬럼 필터·필터 빌더·전역 검색·행 숨김·행 상태/외부 필터    | `externalFilter`, `filterRowVisible`, `hiddenRowIds`                              |
| `paging`         | `pagingFeature`         | 페이지네이션                                               | `pageSize`                                                                        |
| `scrolling`      | `scrollingFeature`      | 가상 스크롤·행 높이·스크롤 요청(`scrollToRow` 등)          | `virtualScroll`, `autoRowHeight`                                                  |
| `virtualColumns` | `virtualColumnsFeature` | 컬럼 가상화 — 뷰포트 안 컬럼만 렌더                        | `virtualColumns`                                                                  |
| `grouping`       | `groupingFeature`       | 행 그룹화·소계·트리 데이터 표시·펼침 상태                  | `treeData`, `groupSubtotals`                                                      |
| `pivot`          | `pivotFeature`          | 피벗 테이블(행/열 디멘션 + 집계)                           | `pivot`                                                                           |
| `serverSide`     | `serverSideFeature`     | 서버사이드 행 모델·Append Scroll·서버 그룹/피벗 지연 로드  | `serverSide`, `appendScroll`                                                      |
| `selection`      | `selectionFeature`      | 행 선택·전체 선택·조건부 선택                              | `isRowSelectable`                                                                 |
| `cellSelection`  | `cellSelectionFeature`  | 활성 셀·범위 선택·셀 내비게이션                            | `selectionMode`                                                                   |
| `editing`        | `editingFeature`        | 셀 편집·에디터·채우기 핸들·범위 이동/지우기·체크박스 컬럼  | `editable`, `singleClickEdit`, `editType`, `beforeEdit`, `fillSeries`             |
| `clipboard`      | `clipboardFeature`      | TSV 복사/붙여넣기·전처리 훅                                | `beforePaste`, `beforeCopy`, `clipboardDelimiter`, `pasteExtend`, `copyFormatted` |
| `rows`           | `rowsFeature`           | 행 CRUD·행 상태(I/U/D)·변경분 수집                         | —                                                                                 |
| `history`        | `historyFeature`        | Undo/Redo 이력                                             | `undoLimit`                                                                       |
| `validation`     | `validationFeature`     | 셀 검증·수동 에러                                          | —                                                                                 |
| `detail`         | `detailFeature`         | 행 상세 패널(master-detail)                                | —                                                                                 |
| `find`           | `findFeature`           | 셀 찾기/바꾸기                                             | —                                                                                 |
| `pinned`         | `pinnedFeature`         | 상단/하단 고정 행                                          | `pinnedTopRows`, `pinnedBottomRows`                                               |
| `rowDrag`        | `rowDragFeature`        | 행 드래그 재정렬·그리드 간 행 이동                         | `rowDrag`, `rowDragAcceptExternal`                                                |
| `styling`        | `stylingFeature`        | 행/셀 클래스·스타일·줄무늬·셀 플래시                       | `rowClass`, `rowStyle`, `striped`, `cellFlash`                                    |
| `export`         | `exportFeature`         | CSV/XLSX/PDF 보내기·가져오기·인쇄                          | —                                                                                 |
| `contextMenu`    | `contextMenuFeature`    | 셀/헤더 컨텍스트 메뉴·셀 툴팁·노트                         | `contextMenu`, `headerContextMenu`                                                |
| `fullWidth`      | `fullWidthFeature`      | 전체 너비 행(배너·구분선)                                  | `isFullWidthRow`, `fullWidthRenderer`                                             |
| `animateRows`    | `animateRowsFeature`    | 행 이동 애니메이션(FLIP) — 활성 플래그 소유, 적용은 렌더러 | `animateRows`                                                                     |

`allFeatures()`는 위 25개 전체 프리셋을 반환하고, `lazyFeature(name, loader)`는
비동기 기능을 등록하는 헬퍼다(아래 참조).

기능 간 의존에 주의 — 예를 들어 `clipboard`는 `editing`·`validation`·`rows`를
`ctl()`로 호출하므로, `clipboard`만 등록하면 붙여넣기 시 편집/검증/행 추가가
모두 no-op이 된다. 어댑터 UI(필터 드롭다운, 컨텍스트 메뉴 등)도 연결된
기능이 꺼져 있으면 조용히 동작하지 않으니, 사용할 사용자 시나리오를 기준으로
기능 집합을 고른다.

## 커스텀 기능 (GridFeature)

```ts
import type { GridFeature, GridController } from "@moda-grid/core";

const bannerFeature: GridFeature<User> = {
  name: "banner",
  create: (host, options) => ({
    name: "banner",
    // 페이징까지 적용된 표시 행을 변환하는 마지막 훅 (선택)
    applyPipeline: (rows) => [...rows, makeBannerRow()],
    // 스냅샷·영속 조각 (선택) — 등록표 순회에 자동 참여
    snapshotSlice: () => ({ loading: false }),
    exportState: () => ({ bannerOn: true }),
    importState: (state) => {},
    reset: () => {},
    dispose: () => {},
  }),
};

const grid = new GridCore({ columns, data, features: [bannerFeature] });
```

| `GridController` 멤버          | 역할                                                        |
| ------------------------------ | ----------------------------------------------------------- |
| `name` (필수)                  | 기능 식별자 — 등록표 키                                     |
| `applyPipeline?(rows)`         | 페이징 후·표시 확정 전 행 배열 변환 훅 — 등록 순서대로 실행 |
| `snapshotSlice?()`             | `getSnapshot()`에 기여하는 필드 조각                        |
| `exportState?()`/`importState` | `getState()`/`applyState()` 영속 조각 (자유로운 키 허용)    |
| `reset?()`/`dispose?()`        | 리셋·파괴 훅                                                |

컨트롤러 안에서 다른 기능은 `host.internals.ctl("sorting")`처럼 이름으로
조회한다. 위 표의 `name`이 조회 키 — 내장 이름은 정확한 컨트롤러 타입,
외부 이름은 `GridController | undefined`를 돌려준다. 미등록 내장 기능은
inert 스텁이 반환돼 호출은 안전하지만 실질 no-op이다.

## `create(host, options)`에서 쓸 수 있는 표면

커스텀 컨트롤러는 `GridCore` 클래스가 아니라 `GridHost` 인터페이스만
의존한다 — 커널과의 모든 소통은 이 창구를 통한다.

### `GridHost` — 커널 데이터 + 공개 API

| 멤버                                               | 설명                                                    |
| -------------------------------------------------- | ------------------------------------------------------- |
| `host.grid`                                        | facade 인스턴스 — `(grid) => ...` 사용자 콜백에 넘길 때 |
| `host.rawData`                                     | 원본 데이터 — 교체 가능(CRUD 기능이 쓰는 경로)          |
| `host.visibleData`                                 | 파이프라인 출력 행 (읽기 전용)                          |
| `host.columns`                                     | 컬럼 정의 원본 — 상태 미반영                            |
| `host.locale`                                      | 병합된 로케일                                           |
| `host.getRowId(row)`                               | 행 ID 해석 (`getRowId` 옵션 또는 자동 ID)               |
| `host.getCellValue(row, col)` / `getCellText`      | valueGetter/formula 해석 값 / 포맷된 표시 문자열        |
| `host.getRowById(id)` / `getDisplayedRowIndex(id)` | 행 조회 / 표시 인덱스                                   |
| `host.getColumn(field)`                            | field로 컬럼 정의 조회                                  |
| `host.getActiveCell()` / `setActiveCell`           | 활성 셀 위치 조회·지정                                  |
| `host.validateCellValue(row, col, value)`          | 컬럼 required/validate 검증 — 에러 문자열 또는 null     |
| `host.setData(data)`                               | 원본 데이터 교체 (이력·행 상태 초기화)                  |
| `host.getPageSize()`                               | 페이지 크기 — 0이면 페이징 비활성                       |
| `host.notify()`                                    | 파이프라인 재계산 + 구독자 통지 — 상태 바꾼 뒤 호출     |
| `host.batch(fn)`                                   | 여러 변경을 통지 1회로 묶기                             |

### `host.internals` — 형제 조회 + 커널 채널

| 멤버                                                                     | 설명                                                             |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| `ctl(name)`                                                              | 형제 컨트롤러 조회 — 위 표의 `name` 키 사용                      |
| `invalidate()`                                                           | 파이프라인 재계산 없이 스냅샷만 무효화 + 통지 (표시 상태 변경용) |
| `emitEvent(type, payload)`                                               | `grid.on` 구독자에게 타입 이벤트 발행                            |
| `recordEdits(edits, source?)`                                            | 셀 변경분을 Undo 이력에 기록 + 행 상태 갱신                      |
| `readValue(row, col)` / `getCellValueAt(r, c)` / `getColumnIndex(field)` | 값·좌표 브리지                                                   |
| `findRowDeep(id)`                                                        | rawData + 트리 자식까지 DFS로 행 탐색                            |
| `serverSide` / `rtl`                                                     | 서버 모드 / RTL 모드 플래그                                      |
| `isDisplayOrderMutable()`                                                | 표시 순서가 rawData 순서와 같은지 — 로컬 행 재정렬 가능 여부     |
| `replaceColumns(cols)` / `setVisibleData(rows)`                          | 커널 소유 컬럼/표시 데이터 교체 채널                             |
| `allSelectableRows()`                                                    | 전체선택 대상 행 (필터 통과 전체, 페이징 전)                     |
| `treeOpts` / `treeSelectsChildren` / `hasCustomGetRowId`                 | 커널 옵션 뷰                                                     |
| `refreshPipeline()`                                                      | 통지 없이 파이프라인만 재계산 — 붙여넣기 행 확장 등 중간 단계용  |
| `filteredRows`                                                           | 필터·검색 통과 후 정렬/페이징 전 행                              |

규칙:

- 커스텀 컨트롤러도 `snapshotSlice()`·`exportState()`·`importState()`·
  `reset()`·`dispose()`를 구현하면 스냅샷·`getState`/`applyState`·
  `resetView`·`destroy` 순회에 자동으로 참여한다.
- 반환하는 스냅샷/영속 조각은 **새 객체**여야 한다 — 내부 Set·Map·배열
  참조를 그대로 내면 스냅샷 비교와 상태 복원이 오염된다.
- 상태를 바꾼 뒤에는 `host.notify()`(파이프라인 영향 있음) 또는
  `host.internals.invalidate()`(표시 상태만)를 호출해야 어댑터가 갱신된다.
- `name`은 커널 등록표의 키다 — `GridFeature.name`과 `create()`가 돌려주는
  컨트롤러의 `name`이 다르면 등록되지 않는다.

## v1 → v2: 확장 경로가 서브클래싱에서 기능 등록으로

v2에서 `GridCore`의 `protected` 멤버는 전부 제거·내부화됐다 —
**서브클래싱은 더 이상 확장 경로가 아니다.** v1에서 서브클래스로
커스터마이즈하던 코드는 다음처럼 옮긴다:

- 커스텀 동작 추가 → `GridFeature`로 만들어 `features`에 등록한다.
- 서브클래스의 `this.sorts`, `this.filters` 같은 protected 접근자 →
  `host.internals.ctl("sorting").specs`처럼 컨트롤러를 이름으로 조회한다.
- 내장 기능의 동작 일부를 바꾸던 서브클래스 → 같은 `name`의 팩토리로
  **덮어쓴다**(아래) — 커널이 요구하는 멤버 표면 검증을 통과해야 한다.
- `grid.internals`는 외부 코드에서도 같은 조회를 제공한다 —
  `grid.internals.ctl("sorting")`로 컨트롤러 공개 멤버에 접근할 수 있다.

## 비동기(lazy) 기능

`create`가 `Promise<GridController>`를 반환하면 지연 로드 기능으로
등록된다. `lazyFeature` 헬퍼로 동적 import와 연결하면 해당 기능
코드가 별도 청크로 분리된다:

```ts
import { GridCore, lazyFeature, columnsFeature } from "@moda-grid/core/slim";

const grid = new GridCore({
  columns,
  data,
  features: [
    columnsFeature(),
    lazyFeature("pivot", () =>
      import("./pivot-feature.js").then((m) => m.pivotFeature()),
    ),
  ],
});
```

- 해결 전까지 inert 스텁이 슬롯을 채운다 — 공개 API 호출은 no-op이고
  (미등록 경고는 나지 않는다), 스냅샷·영속 필드는 빈 값으로 유지된다.
- 해결되면 실제 컨트롤러가 같은 슬롯을 대체하고 커널이 자동으로
  `notify()`한다 — 초기 옵션 상태와 `applyPipeline` 훅도 이때 반영된다.
- 해결 전 `applyState()`로 들어온 해당 기능의 영속 조각은 버퍼링되어
  해결 시 재생된다.
- 로드 실패 시 개발 경고를 내고 기능은 비활성 상태로 남는다.
- `destroy()` 후 해결된 컨트롤러는 즉시 `dispose()`되고 통지하지 않는다.

## 내장 기능 덮어쓰기

`features`에 내장과 같은 `name`의 기능을 넣으면 팩토리를 덮어쓴다.
덮어쓴 컨트롤러는 커널이 요구하는 멤버 표면을 모두 구현해야 한다 —
빠진 멤버가 있으면 생성 시점에 `TypeError`로 즉시 실패한다.
`snapshotSlice`/`exportState`를 구현하지 않으면 내장 기본값이
스냅샷·영속 폴백으로 유지된다.

## 슬림 엔트리와 트리셰이킹

```ts
import { GridCore } from "@moda-grid/core/slim";
import { columnsFeature, sortingFeature } from "@moda-grid/core/slim";

const grid = new GridCore({
  columns,
  data,
  features: [columnsFeature(), sortingFeature()],
});
```

- 슬림 `GridCore`는 `features` 미지정 시 **모든 내장 기능이 비활성**이다
  — 명시한 것만 번들에 남는다.
- 기본 엔트리(`@moda-grid/core`)의 `GridCore`는 미지정 시 전체 기능을
  주입한다 — 기존 코드와 완전히 호환된다.
- 측정 예시: columns+sorting만 번들하면 ~179KB, 전체 기능은 ~4.3MB
  (esbuild 기준, xlsx·pdf 등 서드파티 포함).
- 어댑터(`@moda-grid/react` 등)는 풀 기능 기준으로 렌더링하므로, 슬림
  그리드를 어댑터에 넘길 때는 활성화하지 않은 기능의 UI가 no-op으로
  동작함을 감안한다.
