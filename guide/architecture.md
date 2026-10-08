# 아키텍처 — 커널/컨트롤러 모듈화 가이드

`GridCore`의 내부 구조와 기능(feature) 확장 방법을 다루는 **개발자 가이드**다.
기능 사용법은 각 `features/*.md`를, 문서 추가 규칙은
[extending.md](./extending.md)를 참고한다.

핵심 소스 위치:

| 역할                | 경로                                                        |
| ------------------- | ----------------------------------------------------------- |
| 공개 facade / 커널  | `packages/core/src/grid.ts` (`GridCore`)                    |
| 계약 인터페이스     | `packages/core/src/kernel/host.ts`                          |
| 내장 컨트롤러       | `packages/core/src/features/*.ts`                           |
| 기능 팩토리         | `packages/core/src/features/index.ts`                       |
| 전체 엔트리         | `packages/core/src/gridFull.ts` → `@moda-grid/core`         |
| 슬림 엔트리         | `packages/core/src/slim.ts` → `@moda-grid/core/slim`        |
| 5렌더러 공유 계약   | `packages/core/src/conformance.ts`                          |

## 1. 전체 구조

```
                    ┌──────────────────────────────┐
   어댑터 5종 ────► │         GridCore (facade)     │  packages/core/src/grid.ts
 React/Vue3/Vue2/   │  · 커널 상태: data/columns/   │
 Svelte/mountGrid   │    옵션/파이프라인/이벤트     │
                    │  · public API 전용 — protected│
                    │    멤버 없음                  │
                    └──────────┬───────────────────┘
                               │ GridHost (커널이 구현)
                               │  · rawData/visibleData/columns/locale
                               │  · notify()/batch()/setData() ...
                               │ GridInternals
                               │  · ctl()/invalidate()/emitEvent()/
                               │    readValue()/refreshPipeline() ...
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      FilteringController  SortingController  SelectionController ...
      (features/filtering) (features/sorting) (features/selection)
              │
              └── GridController 규약: name / snapshotSlice? /
                  exportState? / importState? / reset? / dispose? /
                  applyPipeline?
```

역사적으로 `grid.ts`는 모든 기능 로직이 뭉친 약 8,000줄의 God class였다.
모듈화는 두 계약으로 이를 분해했다:

- **커널(`GridCore`)** — 데이터·컬럼·옵션·파이프라인·이벤트·스냅샷 조립과
  공개 API(facade)만 소유한다. `protected` 멤버는 없다 — 컨트롤러는
  protected 브리지가 아니라 `GridHost`/`GridInternals`를 통해 커널에 접근한다.
- **기능 컨트롤러** — `features/*.ts`의 독립 클래스. 각자 자기 상태
  (예: `FilteringController.filters`)와 로직을 소유하고, 공통 선택 규약
  `GridController`를 구현한다. 테스트에서 커널 없이 단독 생성할 수 있다.

## 2. 핵심 인터페이스 (`kernel/host.ts`)

### `GridController<TData>` — 모든 컨트롤러의 선택 규약

```ts
export interface GridController<TData extends object = RowData> {
  readonly name: string;                        // 등록표 키와 동일해야 함
  snapshotSlice?(): Partial<GridSnapshot<TData>>;  // getSnapshot 조립 기여분
  exportState?(): object;                        // getState 영속화 조각
  importState?(state: Partial<GridPersistedState>): void;
  reset?(): void;                                // resetView 시 초기화
  dispose?(): void;                              // destroy 시 자원 해제
  applyPipeline?(rows: TData[]): TData[];        // 커스텀 파이프라인 훅
}
```

전부 선택 사항이다 — 필요한 멤버만 구현한다.

### `GridFeature<TData>` — 기능 등록 단위

```ts
export interface GridFeature<TData extends object = RowData> {
  readonly name: string;   // 생성 컨트롤러의 name과 동일해야 함
  create(
    host: GridHost<TData>,
    options: GridOptions<TData>,
  ): GridController<TData> | Promise<GridController<TData>>;
}
```

`Promise` 반환은 비동기(lazy) 기능 — §5 참조.

### `GridHost<TData>` — 컨트롤러 → 커널 창구

`GridCore`가 직접 구현한다. 커널 데이터의 읽기 뷰(`rawData`,
`visibleData`, `columns`, `locale`)와 이미 public인 커널 API
(`getRowId`, `getCellValue`, `setData`, `notify`, `batch` 등)를 노출한다.
컨트롤러가 **사용자와 같은 표면**으로 커널을 쓰도록 설계됐다 — 컨트롤러가
호출할 수 있는 것은 사용자도 호출할 수 있다.

### `GridInternals<TData>` — 형제 조회 + 커널 브리지

- `ctl("filtering")` — 형제 컨트롤러 조회. `GridControllerMap`에 등록된
  내장 이름은 정확한 타입으로, 외부 이름은 `GridController | undefined`로
  반환된다. 미등록 내장 기능은 **inert 스텁**(§4)이 반환돼 호출이 안전하다.
- 커널 채널 — `invalidate()`, `emitEvent()`, `recordEdits()`,
  `readValue()`, `refreshPipeline()`, `filteredRows`,
  `replaceColumns()`, `serverSide`, `rtl` 등.

### `GridControllerMap<TData>` — 이름 → 타입 맵

`columns`, `sorting`, `filtering`, `paging`, `scrolling`, `virtualColumns`,
`grouping`, `pivot`, `serverSide`, `selection`, `cellSelection`, `editing`,
`clipboard`, `rows`, `history`, `validation`, `detail`, `find`, `pinned`,
`rowDrag`, `styling`, `export`, `contextMenu`, `fullWidth`, `animateRows`.
새 내장 기능을 추가하면 이 맵에도 키를 등록한다.

## 3. 소유권 규칙 — 무엇이 커널이고 무엇이 컨트롤러인가

| 소유 | 내용 |
| ---- | ---- |
| 커널 | `rawData`/`visibleData`, 컬럼 배열, 옵션 해석, 행 파이프라인 순서, 스냅샷 조립, 이벤트 버스, 구독(`subscribe`), 공개 API facade, ID 해석 |
| 컨트롤러 | 기능 상태(`filters`, `sortModel`, 선택 범위, 편집 세션, 드래그 세션…), 기능 로직, 해당 기능의 파이프라인 단계(정렬 적용 등), 스냅샷 조각, 영속화 조각 |

실무 규칙:

- **컨트롤러 ↔ 컨트롤러**는 `internals.ctl("name")`로 조회한다 —
  직접 참조 저장·전역 싱글턴·`GridCore` 내부 접근 금지.
- **컨트롤러 → 커널**은 `host.*`(공개 API와 동일 표면)와
  `host.internals.*`(커널 채널)만 쓴다. `GridCore`의 private 내부를
  캐스트로 우회하지 않는다 — protected 브리지는 제거됐다.
- **커널 → 컨트롤러**는 `GridController` 규약 멤버와 커널이 아는 기능별
  호출점(파이프라인 단계 호출)을 통한다.
- 공개 API는 facade에 둔다 — 사용자가 `grid.setFilter()`를 부르면
  `GridCore`가 `ctl("filtering").setFilter()`에 위임한다. 컨트롤러
  메서드 시그니처는 facade와 동일하게 유지하면 위임이 단순하다.

## 4. 기능 등록과 inert 스텁

```ts
// 기본(전체 엔트리) — features 미지정 시 allFeatures() 전체 활성
import { GridCore } from "@moda-grid/core";
const grid = new GridCore({ columns, data });

// 슬림 엔트리 — 명시한 기능만 활성
import { GridCore, sortingFeature, pagingFeature } from "@moda-grid/core/slim";
const slim = new GridCore({
  columns,
  data,
  features: [sortingFeature(), pagingFeature()],
});
```

핵심 계약:

- **`features` 미지정 시 커널 기본값은 "전부 비활성"** — `grid.ts`의
  `GridCore`(커널)는 컨트롤러 클래스를 정적 import하지 않는다.
  전체 활성 래핑은 `gridFull.ts`(=`@moda-grid/core` 기본 엔트리)가
  `features ?? allFeatures()`로 주입한다. 슬림 엔트리에서 커널이
  컨트롤러를 알아버리면 tree-shaking이 깨진다.
- **inert 스텁** — 등록되지 않은 내장 기능 이름도 `ctl()`이 객체를
  반환한다. 모든 메서드가 no-op이고 `snapshotSlice()`는 빈 객체를
  기여하므로, 형제 컨트롤러가 "기능 없음"을 분기 없이 처리할 수 있다.
  같은 이유로 `features`에 없는 기능의 공개 API도 호출은 되지만 no-op이다.
- **`name` 일치** — `GridFeature.name`과 `create()`가 만드는
  컨트롤러의 `name`이 같아야 등록표가 연결된다. 사용자 `features`에
  내장과 같은 `name`을 넘기면 **내장 팩토리를 덮어쓴다**(커스텀
  구현으로 교체하는 공식 경로).

커스텀 기능 예시:

```ts
import type { GridFeature, GridHost, GridOptions } from "@moda-grid/core/slim";

const rowCounter = <T extends object>(): GridFeature<T> => ({
  name: "rowCounter",
  create: (host: GridHost<T>) => ({
    name: "rowCounter",
    // 스냅샷에 커스텀 필드 기여 — 어댑터는 snapshot.rowCounter로 읽는다
    snapshotSlice: () => ({ rowCount: host.visibleData.length }),
    applyPipeline: (rows) => rows, // 변환 없이 통과도 가능
    dispose: () => {},
  }),
});
```

## 5. Lazy 기능 (비동기 `create`)

`lazyFeature`로 기능을 동적 import로 미룬다:

```ts
new GridCore({
  features: [
    ...allFeatures(),
    lazyFeature("pivot", () =>
      import("@moda-grid/core/features/pivot.js").then((m) =>
        m.pivotFeature(),
      ),
    ),
  ],
});
```

커널 동작 순서:

1. `create()`가 Promise를 반환하면 해당 슬롯은 **inert 스텁**으로 채운다 —
   공개 API no-op, 미등록 경고 없음, 스냅샷·영속 계약 유지.
2. 해결되면 커널이 실제 컨트롤러로 교체한다: `controllerMap` 교체 →
   `ctl()` 조회·필드 프록시가 즉시 실제 인스턴스를 가리킴, 스텁 자리에
   삽입해 스냅샷 조각 우선순위 유지, `applyPipeline` 수집,
   해결 전 호출된 `importState` 재생.
3. 후처리 — `serverSide`면 `autoLoad()`, `columns`면 `withDefaults`
   재적용 + `sync()`, 마지막에 파이프라인 재계산 + `notify`.
4. 로드 실패 시 devWarn 후 inert 유지. `destroy()` 후 해결되면 즉시
   `dispose()`하고 아무 통지도 하지 않는다.

**lazy 기능 작성 주의점**: 파이프라인 참여 기능(lazy 대기 중 스텁은
"미준비")은 커널이 `isFeatureReady`(등록 + 해결 완료)로 게이트한다 —
스텁의 no-op `applyPostSort` 같은 메서드가 `undefined`를 반환해 행을
오염시키는 함정이 있다. 기능 단계 호출이 있는 커널 경로는 항상 이 게이트를
거친다.

## 6. 파이프라인과 스냅샷

행 파이프라인(`refresh()`)의 큰 순서:

```
rawData → treeData 평탄화 → filtering.apply(컬럼필터/숨김/상태/검색/외부)
  → grouping(그룹 트리/소계) → sorting.applyPostSort → paging
  → applyPipeline 훅(features 등록 순) → visibleData
```

- `snapshotSlice()` — 각 컨트롤러가 `getSnapshot()`에 기여하는 읽기
  조각. **반환값은 새 객체/복사본이어야 한다** — 스냅샷은 변경 감지의
  기준이라 내부 참조를 공유하면 오염된다.
- `exportState()`/`importState()` — `getState`/`applyState` 영속화.
  배열·Set은 복사해서 넣고 복원 시에도 복사해 분리한다.
- `reset()` — `resetView()`·데이터 교체에서 호출된다. **외부 세션·타이머·
  임시 플래그까지 정리해야 한다** — 재계산 경로마다 호출될 수 있으므로
  상태만 초기화하고 부수 효과는 남기지 않는다.
- `dispose()` — `destroy()`에서 호출. document/window 리스너, 타이머,
  모듈 공유 세션을 해제한다.

## 7. 어댑터 계약 — 5개 렌더러는 얇은다

React/Vue3/Vue2/Svelte/`mountGrid`는 모두 같은 규칙을 따른다:

- 기능 로직·상태는 코어가 소유. 어댑터는 `snapshot`을 읽어 DOM을 만들고
  이벤트를 코어 API(`grid.setFilter`, `navigateCell` …)로 번역한다.
- 매칭·정규화 같은 **판정 로직을 어댑터에 복제하지 않는다** — 공용 헬퍼
  (`matchFilterValue`, `buildColumnFilter`, `computeVirtualColumns`,
  `domToLogicalScrollLeft` …)를 코어에서 export해 공유한다.
- DOM 계약(클래스·data 속성·이벤트)은 5개 렌더러가 동일해야 한다 —
  `.mg-header-filter-btn`, `.mg-filter-drop-panel`, `.mg-filter-cond-*`,
  `td[data-mg-row][data-mg-col]` 등.
- prop → 코어 동기화 이펙트가 호출하는 코어 setter는 **같은 입력에
  멱등**이어야 한다 — prop 참조가 매 렌더 바뀌는 프레임워크에서
  notify → 재렌더 → 재동기화 무한 루프를 막는다.

구체 예: 필터 빌더는 코어가 `ColumnFilter`(`operator`+`value`/`valueTo` +
`values` + `join`) 모델과 `matchFilterValue`의 AND/OR 결합을 소유하고,
5개 렌더러는 `.mg-filter-cond-op`/`.mg-filter-cond-val`/`.mg-filter-join-radio`
컨트롤을 렌더해 `buildColumnFilter(입력 병합)`을 `setFilter`에 넘길
뿐이다 — 평가 로직은 어디에도 복제돼 있지 않다.

## 8. 컨포먼스 — `packages/core/src/conformance.ts`

`runGridConformance(label, mount)`가 DOM으로 검증 가능한 공용 계약을
한 번 기술하고, 각 어댑터의 `conformance.test.*`가 자기 프레임워크의
마운트 함수(`ConformanceMount`)로 실행한다:

```ts
// packages/react/src/conformance.test.tsx
runGridConformance("react", async (opts) => {
  const el = document.createElement("div");
  document.body.appendChild(el);
  const root = createRoot(el);
  await act(async () => { root.render(createElement(DataGrid, opts)); });
  return { el, flush: () => act(async () => {}), destroy: () => ... };
});
```

새 DOM 계약 추가 절차:

1. `ConformanceMount`의 `opts`에 필요한 옵션 필드를 추가한다 — 각
   어댑터 마운트가 `opts`를 자기 props로 번역한다.
2. `runGridConformance` 안에 `it(...)`로 선택자 + 상호작용 + 기대 DOM을
   기술한다.
3. 5개 패키지의 `pnpm test`로 전 렌더러를 검증한다 — 한쪽 구현이 빠지면
   즉시 실패한다.

테스트 작성 요령(기존 계약이 보증하는 함정):

- React는 onChange가 `input`/`click`에 매핑된다 — DOM `change`만 보내면
  무반응. 텍스트 입력은 프로토타입 `value` setter로 우회 후
  `input`+`change` 둘 다 발행하고, 체크박스/라디오는 `.click()`을 쓴다.
- React `DataGrid`의 조기 return(슬림 경로) — 패널류 요소가 하나라도
  필요하면 진입 조건에 반영하고, **모든 훅은 조기 return 위에 선언**한다.
  경로 전환 시 훅 수가 달라지면 "Rendered more hooks"로 크래시한다.
- 0행일 때 `.mg-empty` 플레이스홀더 `tr`이 렌더된다 — 행 수를 셀 때는
  `td[data-mg-row]` 기준으로 센다.

## 9. 테스트 전략

| 층 | 대상 | 위치 |
| -- | ---- | ---- |
| 단위 | 순수 헬퍼·컨트롤러(커널 없이) | `packages/core/src/*.test.ts` |
| 컨포먼스 | DOM 계약, 5렌더러 공유 | `conformance.ts` + 각 어댑터 `conformance.test.*` |
| 어댑터 | 바인딩·반응형 스토어 | `packages/*/src/*.test.*` |
| E2E | 실제 브라우저 데모 | `e2e/*.spec.ts` |

컨트롤러는 `GridHost`를 스텁으로 만들어 단독 인스턴스 테스트가 가능하고,
파사드 레벨 검증은 `new GridCore({...})`로 한다.

## 10. 새 기능 추가 체크리스트

1. 타입 — `types.ts`에 옵션/스냅샷/상태 필드 추가
2. 컨트롤러 — `features/<name>.ts` (상태·로직·스냅샷/영속 조각)
3. 팩토리 — `features/index.ts`에 `<name>Feature()` 추가 +
   `allFeatures()` 순서 반영, `GridControllerMap`에 키 추가
4. facade — 필요 시 `grid.ts`에 공개 API 위임 메서드
5. 헬퍼 — 매칭/정규화 등 순수 로직은 `filtering.ts`류 순수 모듈에 두고
   `index.ts`·`slim.ts` export 확인
6. 5개 어댑터 — 동일 DOM 계약으로 렌더/이벤트 번역만
7. 컨포먼스 — `runGridConformance`에 계약 추가
8. 데모 — `apps/dev-*` FEATURES + 실제 옵션 활성화
9. E2E — `e2e/` 해당 스펙
10. 문서 — `docs/guide/features/*.md` + README 색인 + migration 표
11. 검증 — `pnpm test && pnpm typecheck && pnpm build && pnpm build:apps`

## 11. 흔한 실수

- 로직을 한 어댑터에만 넣고 나머지를 잊는다 — 컨포먼스가 잡도록 계약부터 쓴다
- `GridCore` 내부를 캐스트로 우회한다 — 필요한 통로는 `GridInternals`에 추가
- prop 동기화 setter가 같은 입력에도 notify/서버 캐시 purge를 한다 — 멱등 가드
- `reset()`/`dispose()`에서 세션·리스너·타이머를 남긴다
- `snapshotSlice()`/`exportState()`가 내부 배열·Set·Map 참조를 그대로 낸다
- lazy 스텁의 no-op 반환값이 파이프라인 중간 결과를 오염시킨다 —
  `isFeatureReady` 게이트로 커널이 걸러주지만, 기능 단계 호출을 새로 만들면
  같은 게이트를 쓴다
- `setFilter`류가 서버사이드 `purge()`·페이징 리셋을 빼먹는다 — 기존 컨트롤러의
  호출 패턴을 따라야 한다

## 12. 공개 API·스냅샷 호환 원칙

- `GridOptions`·`ColumnDef`·`GridSnapshot`·`ColumnFilter` 등 공개 타입의
  기존 필드 의미는 바꾸지 않는다 — 확장은 선택 필드 추가로 한다
  (예: `ColumnFilter`에 `values`/`join` 추가, 기존 단일 조건 모델은 그대로 유효).
- `getFilterModel()`/`setFilterModel()`/`getState()` 등 모델 계약은
  직렬화 가능하고 깊은 복사로 주고받는다 — 호출자가 반환값을 변형해도
  내부 상태가 오염되면 안 된다.
- 새 기능의 기본값은 "꺼짐/기존 동작"을 유지한다 — 기존 앱이 옵션 없이
  업그레이드해도 동작이 바뀌지 않는다.
