# 계산 컬럼 — 수식(formula) / 누계(cumulative)

실제 데이터에 없는 파생 값을 컬럼으로 표시한다. IBSheet의 Formula와 누계에
해당한다.

## 수식 컬럼 (`ColumnDef.formula`)

컬럼 `field`명을 변수로 쓰는 JS 표현식으로 행 단위 자동 계산:

```ts
const columns: ColumnDef<Order>[] = [
  { field: "price", header: "단가" },
  { field: "qty", header: "수량" },
  {
    field: "total",
    header: "합계",
    formula: "price * qty",                    // 다른 필드를 변수로 참조
    format: { type: "currency" },              // 내장 포맷터와 조합 가능
    aggregationFn: "sum",                      // 그룹 집계에도 반영됨
  },
  { field: "taxed", header: "세금포함", formula: "Math.round(price * qty * 1.1)" },
];
```

- 표현식은 JS 문법 그대로 — `Math.round(price*1.1)`, 문자열 결합
  `name + ' (' + id + ')'` 등이 가능하다. 컬럼 field와 일치하는 식별자만
  `row["field"]`로 치환되고 `Math`/`Number` 등 전역은 그대로 동작한다.
- **정렬·필터·집계·CSV/xlsx보내기**에 계산값이 반영된다 (`readValue` 경로).
- 계산 컬럼은 **읽기 전용** — 편집(`startEditing`)과 붙여넣기 대상에서
  제외된다.
- 평가 오류(문법·참조 실패)와 `NaN` 결과는 `null`로 표시된다.
- 값은 행에 저장되지 않고 매번 계산된다 — 기준 필드 편집 시 즉시 갱신.
- `valueGetter`가 함께 있으면 valueGetter가 우선한다.

```ts
import { evaluateFormula } from "@moda-grid/core";
evaluateFormula("price * qty", row, ["price", "qty"]); // 직접 평가
```

## 누계 컬럼 (`ColumnDef.cumulative`)

지정 필드의 값을 **표시 순서대로 누적 합산**해 표시한다 (IBSheet 누계):

```ts
const columns: ColumnDef<Sale>[] = [
  { field: "date", header: "일자" },
  { field: "amount", header: "금액" },
  { field: "cum", header: "누계", cumulative: "amount", format: { type: "number" } },
];
// 10 → 30 → 60 → 100 처럼 표시
```

- 정렬·필터가 바뀌면 **표시 순서 기준으로 누계를 재계산**한다.
- 누계 컬럼으로 정렬하면 원본 필드(`amount`) 값 기준으로 정렬된다
  (누계값 자체는 순서 의존이라 정렬 기준으로 부적합).
- `field`는 행 타입의 키여야 한다 — 누계 전용 표시 컬럼이면 인터페이스에
  선택 필드(`cum?: number`)로 선언한다.
