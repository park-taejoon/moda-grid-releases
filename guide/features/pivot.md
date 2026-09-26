# 피벗 테이블 (Pivot)

행/열 디멘션과 측정값으로 데이터를 재구성한다 — 엑셀 피벗 테이블 /
IBSheet 피벗 방식.

## 설정

```ts
const grid = new GridCore({
  columns,
  data,
  pivot: {
    rows: ["team"],                              // 행 디멘션 (피벗 행이 됨)
    columns: ["quarter"],                        // 열 디멘션 (값 조합이 컬럼이 됨)
    values: [{ field: "amount", agg: "sum" }],   // 측정값 (집계 함수 지정)
    showColumnTotals: true,                      // 우측 합계 컬럼 (기본값)
    showRowTotal: true,                          // 하단 합계 행 (기본값)
  },
});
```

어댑터는 `<DataGrid pivot={...} />` prop으로 전달한다 (내부 GridCore 생성
모드에서만 적용 — 제어 모드는 `new GridCore({ ..., pivot })`).

## 결과 예시

| 팀 | Q1 금액 | Q2 금액 | 합계 금액 |
| -- | ------- | ------- | --------- |
| A  | 15      | 20      | 35        |
| B  | 30      | 40      | 70        |
| 합계 | 45    | 60      | 105       |

- 행 디멘션은 원본 컬럼의 `header`/`format`을 그대로 물려받는다.
- 측정값 컬럼 헤더는 `"열조합 라벨 측정값헤더"` 형태 — `values[].header`로
  라벨을 바꿀 수 있다.
- 집계 함수: `sum` `avg` `min` `max` `count`.
- 측정값을 여러 개 두면 열 조합 × 측정값만큼 컬럼이 생긴다.
- 열 디멘션(`columns`)을 생략하면 측정값 컬럼만 있는 요약 테이블이 된다.

## 동작 규칙

- **필터·전역 검색·정렬이 피벗에 먼저 적용**된다 — 필터된 원본 데이터만
  피벗되고, 정렬 순서가 디멘션 조합의 등장 순서가 된다.
- 피벗 컬럼으로 헤더 정렬도 가능하다 — **합계 행은 항상 맨 아래** 고정.
- 피벗 활성 동안 컬럼 정의는 생성 컬럼으로 교체된다 — `setPivot(null)`로
  해제하면 원본 컬럼과 너비/순서 상태가 복원된다.
- CSV/xlsx보내기는 피벗 결과를 출력한다.
- 편집·행 상태(I/U/D)는 피벗 행에는 의미가 없다 — 원본 데이터 편집용으로는
  피벗을 해제하고 사용한다.

## 코어 API

```ts
grid.setPivot({ rows: ["team"], columns: ["quarter"], values: [...] });
grid.setPivot(null);    // 해제 — 원본 컬럼/데이터 복원
grid.isPivot();         // 활성 여부
```

순수 함수로도 사용 가능하다:

```ts
import { buildPivot } from "@moda-grid/core";
const { columns, rows } = buildPivot(rows, opts, sourceColumns, readValue);
```
