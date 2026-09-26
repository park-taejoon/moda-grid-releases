# 서버 사이드 데이터 모델 (무한 스크롤)

로컬 파이프라인을 건너뛰고 필요한 행 블록만 서버에서 가져온다.

## 설정

```ts
import type { ServerSideDataSource } from "@moda-grid/core";

const dataSource: ServerSideDataSource<User> = {
  async getRows({ startRow, endRow, sortModel, filterModel }) {
    const res = await fetch(
      `/api/rows?start=${startRow}&end=${endRow}` +
      `&sort=${encodeURIComponent(JSON.stringify(sortModel))}` +
      `&filter=${encodeURIComponent(JSON.stringify(filterModel))}`,
    );
    return res.json();   // { rows: TData[], lastRowIndex?: number }
  },
};

const grid = new GridCore({
  columns,
  data: [],                                          // 서버 모드에서는 무시
  serverSide: { dataSource, cacheBlockSize: 50 },    // 기본 블록 50행
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
- 로컬 페이징과 행 그룹화는 적용되지 않는다.

## 코어 API

```ts
grid.isServerSide();        // 서버 모드 여부
grid.isRowLoaded(i);        // 해당 인덱스 로드 여부 (스켈레톤 판별)
grid.refreshServerRows();   // 캐시 폐기 + 현재 뷰포트부터 재요청
```

## 스냅샷 필드

`snapshot.serverSide = { loading, rowCount, loadedCount, error }`.
`loading` 동안 어댑터는 스크롤 컨테이너 하단에 `.mg-loading` 인디케이터를
표시한다.

## 서버가 받는 파라미터

```ts
interface ServerSideGetRowsParams {
  startRow: number;                          // inclusive
  endRow: number;                            // exclusive
  sortModel: SortSpec[];                     // 다중 정렬 (priority 순)
  filterModel: Record<string, ColumnFilter>; // 컬럼 필터
}
// 반환: { rows: TData[]; lastRowIndex?: number }
```

서버는 정렬·필터·페이징(slice)을 직접 수행하고 행 배열을 반환한다.
`lastRowIndex`는 전체 행 수를 알 때만 포함한다.

## Append Scroll (누적 로드)

`serverSide`의 블록 캐시 방식과 달리, IBSheet의 Append Scroll처럼
**페이지 단위로 행을 끝에 누적**한다. 로컬 파이프라인(필터/정렬)이 그대로
적용된다 — 서버는 순서대로 페이지만 내려주면 된다.

```ts
const grid = new GridCore({
  columns,
  data: [],
  appendScroll: {
    loadPage: async (page) => {
      const res = await fetch(`/api/rows?page=${page}&size=50`);
      return res.json();             // TData[] — 빈 배열/짧은 페이지면 끝으로 간주
    },
    pageSize: 50,                    // 기본값 50
    startPage: 0,                    // 초기 데이터가 있으면 다음 페이지 번호
    autoLoad: true,                  // 생성 시 첫 페이지 자동 로드 (기본값)
  },
});
```

- **자동 트리거** — 어댑터의 스크롤 핸들러가 `maybeLoadMore`를 호출해
  하단 근접 시 다음 페이지를 요청한다 (`<DataGrid appendScroll={...} />`).
- **수동 트리거** — "더 보기" 버튼에는 `grid.loadMore()`를 연결한다.
- **스냅샷** — `snapshot.appendScroll = { loading, hasMore }`로
  로딩 인디케이터와 버튼 disabled를 연동한다.

```ts
await grid.loadMore();        // 다음 페이지 로드. 요청했으면 true
grid.hasMoreRows();           // 추가 데이터 가능 여부
grid.maybeLoadMore(scrollTop, clientHeight, scrollHeight); // 어댑터 내부용
```

- 이미 로딩 중이거나 데이터 끝이면 중복 요청하지 않는다.
- `pageSize` 미만으로 오거나 빈 배열이 오면 `hasMore`가 false가 되어
  더 요청하지 않는다.
- `serverSide`와 동시 지정할 수 없다 — 누적 방식이면 `appendScroll`,
  블록 캐시 방식이면 `serverSide`를 선택한다.

