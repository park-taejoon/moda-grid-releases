# 서버 사이드 데이터 모델 (무한 스크롤)

로컬 파이프라인을 건너뛰고 필요한 행 블록만 서버에서 가져온다.

## 설정

```ts
import type { ServerSideDataSource } from "@moda-grid/core";

const dataSource: ServerSideDataSource<User> = {
  async getRows({
    startRow,
    endRow,
    sortModel,
    filterModel,
    groupBy,
    groupKeys,
    pivotModel,
  }) {
    const res = await fetch(
      `/api/rows?start=${startRow}&end=${endRow}` +
        `&sort=${encodeURIComponent(JSON.stringify(sortModel))}` +
        `&filter=${encodeURIComponent(JSON.stringify(filterModel))}` +
        `&groupBy=${encodeURIComponent(JSON.stringify(groupBy ?? []))}` +
        `&groupKeys=${encodeURIComponent(JSON.stringify(groupKeys ?? []))}` +
        `&pivot=${encodeURIComponent(JSON.stringify(pivotModel ?? null))}`,
    );
    return res.json(); // { rows: (TData|ServerSideGroupRow)[], lastRowIndex?, secondaryColumns? }
  },
};

const grid = new GridCore({
  columns,
  data: [], // 서버 모드에서는 무시
  serverSide: { dataSource, cacheBlockSize: 50 }, // 기본 블록 50행
  virtualScroll: { rowHeight: 37, viewportHeight: 480 }, // 무한 스크롤은 가상 스크롤과 함께
});
```

어댑터에서는 `serverSide` prop으로 전달한다 — **내부 GridCore 생성
모드에서만** 적용된다 (`<DataGrid columns data serverSide={...} />`).
제어 모드면 `useGridCore({ ..., serverSide })` 옵션으로 넘긴다.

## 동작 규칙

- **블록 캐시** — 뷰포트가 걸치는 미로드 블록만 `getRows`로 요청하고
  `Map<rowIndex, row>`에 캐시한다. `loading`/`loaded`/`error` 블록은
  재요청하지 않는다 (실패 블록은 `refreshServerRows()`로 재시도).
- **rowCount** — `lastRowIndex`가 오면 전체 행 수 확정. 없으면
  `max(요청 끝, 로드 끝) + cacheBlockSize`로 추정해 스크롤 영역을 넓히고,
  요청보다 짧은 블록이 오면 데이터 끝으로 확정한다.
- **정렬/필터/검색 변경** — 서버 파라미터가 바뀌면 캐시를 폐기(purge)하고
  현재 뷰포트부터 재요청한다. purge 후 도착한 이전 세대 응답은 epoch 토큰으로
  자동 폐기된다.
- **미로드 행** — `rows`/`virtualRows`의 미로드 인덱스는 `undefined` —
  어댑터가 `.mg-skeleton-row`(컬럼별 shimmer 바)로 렌더링한다.
- 로컬 페이징은 적용되지 않는다. 그룹화·피벗은 서버가 수행한다
  (아래 서버사이드 그룹화/피벗 참조).

## 서버사이드 그룹화

`groupBy`가 설정되면(`setGroupBy`, groupPanel 드래그 등) 그리드는
플랫 블록 대신 **레벨별 요청**으로 전환한다:

- `groupBy` — 요청에 실리는 그룹 기준 필드 목록.
- `groupKeys` — 요청하는 자식 레벨의 부모 그룹 키 경로. 빈 배열/생략이면
  루트 레벨(최상위 그룹 목록)이다.
- 요청 경로 깊이(`groupKeys.length`)가 `groupBy.length`보다 작으면
  서버는 리프 대신 `ServerSideGroupRow`를 반환한다:

```ts
interface ServerSideGroupRow {
  type: "group";
  key: string; // 부모 경로에 붙는 세그먼트 (그룹 값 문자열)
  field: string; // 그룹 기준 컬럼
  value: unknown; // 표시용 값
  childCount: number; // 펼쳤을 때의 자식 슬롯 수
  aggregates?: Record<string, unknown>; // aggregationFn 컬럼에 표시
}
```

```ts
// 서버 측 의사코드
getRows: async ({ groupBy = [], groupKeys = [], startRow, endRow, ... }) => {
  if (groupKeys.length < groupBy.length) {
    const field = groupBy[groupKeys.length];
    // SELECT field, COUNT(*) ... GROUP BY field → 그룹 행 목록
    return { rows: groupRows(field), lastRowIndex: groupCount };
  }
  // 리프 레벨 — groupKeys 경로를 WHERE로 적용해 자식 행 반환
  return { rows: leafRowsWhere(groupKeys), lastRowIndex };
};
```

- **펼침 = 지연 로드** — 그룹 행을 펼치면(`toggleGroupExpanded`/클릭)
  `childCount`만큼 자식 슬롯이 스켈레톤으로 잡히고, 뷰포트에 걸리는
  자식 블록이 `groupKeys=[...부모키]`로 지연 요청된다. 다시 접어도
  자식 캐시는 유지된다.
- **모델 변경 = 전체 폐기** — `groupBy`/피벗이 바뀌면 모든 레벨 캐시를
  폐기하고 루트부터 다시 요청한다. 정렬/필터 변경 시에도 동일하다.
- **`expandAllGroups`** — 이미 로드된 그룹 행만 펼친다 (미로드 깊이는
  펼침 후 지연 로드된다). `collapseAllGroups`는 모든 펼침을 접는다.

## 서버사이드 피벗

`setPivot`/`pivotPanel`로 피벗을 구성하면 요청에 `pivotModel`이 실린다:

```ts
interface ServerSidePivotModel {
  rows: readonly string[]; // 행 디멘션 필드
  columns: readonly string[]; // 열 디멘션 필드
  values: readonly { field: string; agg?: string }[]; // 측정값
}
```

- 서버는 피벗 결과 행을 `rows`로, **생성 컬럼**을 응답의
  `secondaryColumns: ColumnDef[]`로 반환한다 — 커널이 표시 컬럼을
  자동 교체한다 (로컬 피벗의 컬럼 교체 경로와 동일).
- 커스텀 집계 함수는 직렬화 불가라 `values[].agg`에는 내장 집계 키
  (`"sum"`/`"avg"`/`"count"` 등)만 실린다.
- 피벗 해제(`setPivot(null)`) 시 모델 변경으로 캐시가 폐기되고
  플랫 요청으로 돌아간다.
- 그룹화와 피벗은 동시에 켤 수 있다 — `groupBy`는 리프 레벨의 피벗 행
  안에서만 의미를 가진다(실무에서는 둘 중 하나를 쓰는 것을 권장).

## 코어 API

```ts
grid.isServerSide(); // 서버 모드 여부
grid.isRowLoaded(i); // 해당 인덱스 로드 여부 (스켈레톤 판별)
grid.refreshServerRows(); // 캐시 폐기 + 현재 뷰포트부터 재요청
```

## 스냅샷 필드

`snapshot.serverSide = { loading, rowCount, loadedCount, error }`.
`loading` 동안 어댑터는 스크롤 컨테이너 하단에 `.mg-loading` 인디케이터를
표시한다.

## 로딩 오버레이 (`loading`)

서버 모드의 하단 인디케이터(`.mg-loading`)와 별개로, 그리드 전체를 덮는
반투명 로딩 레이어를 표시할 수 있다 — 클라이언트 모드에서 비동기 데이터
조회 중 입력을 차단할 때 사용한다.

```tsx
// React / Vue / Svelte — prop
<DataGrid loading={isFetching} />;

// vanilla — 옵션 또는 런타임 API
const mounted = mountGrid(el, { columns, data, loading: true });
mounted.setLoading(false); // 로딩 해제
```

- 문구는 `locale.loadingMore`(기본 "불러오는 중…")를 사용한다 — 로케일로
  커스터마이즈 가능.
- 오버레이 클래스는 `.mg-loading-overlay`, 활성 시 루트에
  `.mg-loading-host`가 붙는다 (CSS 커스터마이즈 지점).
- 인쇄(`@media print`)에서는 자동으로 숨겨진다.

## 생명주기 이벤트

블록 요청의 시작/성공/실패를 구독할 수 있다 — 로딩 스피너, 요청 로깅,
에러 토스트에 사용한다 (타사 그리드 `OnSearchEnd`/검색 완료 이벤트 대응).

```ts
grid.on("serverRequest", (e) => showSpinner(e.params));
//   { params: ServerSideGetRowsParams } — getRows에 전달된 값
grid.on("serverResponse", (e) => hideSpinner(e.result.rows.length));
//   { params, result: { rows, lastRowIndex? } }
grid.on("serverError", (e) => toast(e.error));
//   { params, error } — 실패 블록은 "error" 표시되고
//   refreshServerRows()로 재시도할 수 있다
```

- `serverRequest`는 `getRows` 호출 직전에 발행된다.
- 정렬/필터 변경으로 purge된 세대의 늦은 응답은 폐기되며
  `serverResponse`/`serverError`도 발행되지 않는다.
- 초기 뷰포트 요청은 생성자에서 시작되므로, 첫 요청을 잡으려면
  어댑터의 `events` prop으로 등록한다 (생성 시점에 연결됨).

## 서버가 받는 파라미터

```ts
interface ServerSideGetRowsParams {
  startRow: number; // inclusive
  endRow: number; // exclusive
  sortModel: SortSpec[]; // 다중 정렬 (priority 순)
  filterModel: Record<string, ColumnFilter>; // 컬럼 필터
}
// 반환: { rows: TData[]; lastRowIndex?: number }
```

서버는 정렬·필터·페이징(slice)을 직접 수행하고 행 배열을 반환한다.
`lastRowIndex`는 전체 행 수를 알 때만 포함한다.

## Append Scroll (누적 로드)

`serverSide`의 블록 캐시 방식과 달리, 타사 그리드의 Append Scroll처럼
**페이지 단위로 행을 끝에 누적**한다. 로컬 파이프라인(필터/정렬)이 그대로
적용된다 — 서버는 순서대로 페이지만 내려주면 된다.

```ts
const grid = new GridCore({
  columns,
  data: [],
  appendScroll: {
    loadPage: async (page) => {
      const res = await fetch(`/api/rows?page=${page}&size=50`);
      return res.json(); // TData[] — 빈 배열/짧은 페이지면 끝으로 간주
    },
    pageSize: 50, // 기본값 50
    startPage: 0, // 초기 데이터가 있으면 다음 페이지 번호
    autoLoad: true, // 생성 시 첫 페이지 자동 로드 (기본값)
  },
});
```

- **자동 트리거** — 어댑터의 스크롤 핸들러가 `maybeLoadMore`를 호출해
  하단 근접 시 다음 페이지를 요청한다 (`<DataGrid appendScroll={...} />`).
- **수동 트리거** — "더 보기" 버튼에는 `grid.loadMore()`를 연결한다.
- **스냅샷** — `snapshot.appendScroll = { loading, hasMore }`로
  로딩 인디케이터와 버튼 disabled를 연동한다.

```ts
await grid.loadMore(); // 다음 페이지 로드. 요청했으면 true
grid.hasMoreRows(); // 추가 데이터 가능 여부
grid.maybeLoadMore(scrollTop, clientHeight, scrollHeight); // 어댑터 내부용
```

- 이미 로딩 중이거나 데이터 끝이면 중복 요청하지 않는다.
- `pageSize` 미만으로 오거나 빈 배열이 오면 `hasMore`가 false가 되어
  더 요청하지 않는다.
- `serverSide`와 동시 지정할 수 없다 — 누적 방식이면 `appendScroll`,
  블록 캐시 방식이면 `serverSide`를 선택한다.
