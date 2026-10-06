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

`features/index.ts`에서 export — `columnsFeature`, `sortingFeature`,
`filteringFeature`, `pagingFeature`, `scrollingFeature`, `groupingFeature`,
`pivotFeature`, `serverSideFeature`, `selectionFeature`,
`cellSelectionFeature`, `editingFeature`, `clipboardFeature`, `rowsFeature`,
`historyFeature`, `validationFeature`, `detailFeature`, `findFeature`,
`pinnedFeature`, `rowDragFeature`, `stylingFeature`, `exportFeature`,
`contextMenuFeature`. `allFeatures()`는 전체 프리셋을 반환한다.

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
조회한다. 내장 이름은 정확한 컨트롤러 타입, 외부 이름은
`GridController | undefined`를 돌려준다.

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
