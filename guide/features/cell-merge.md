# 셀 병합 (Cell Merge)

같은 값의 연속 셀을 `rowSpan`/`colSpan`으로 병합한다. IBSheet의
행 병합/auto-merge에 해당한다.

## 컬럼 옵션

```ts
const columns = [
  { field: "team",  merge: "row" },  // 세로 방향 auto-merge (rowSpan)
  { field: "name" },                 // 병합 안 함 (기본값 "none")
  { field: "q1",    merge: "col" },  // 가로 방향 (colSpan)
  { field: "q2",    merge: "both" }, // 두 방향 모두
];
```

| merge | 동작 |
| ----- | ---- |
| `"row"` | 같은 값의 **연속 행**을 수직 병합 (rowSpan) |
| `"col"` | 같은 값의 **연속 열**을 수평 병합 (colSpan) |
| `"both"` | 두 방향 모두 적용 |
| `"none"` | 병합 안 함 (기본값) |

## 동작 규칙

- **연속 구간만** 병합된다 — 값이 바뀌거나 병합 안 하는 컬럼을 만나면 경계.
- 정렬·필터·페이징 등 **파이프라인 재계산 시 span도 재계산**된다 — 정렬로
  같은 값이 멀어지면 병합이 풀리고, 모이면 다시 합쳐진다.
- merge 미설정 컬럼은 절대 병합에 참여하지 않는다.
- 그룹화/트리/서버 사이드 모드에서는 적용되지 않는다(표시 행이
  단순 목록일 때만).

## 코어 API

```ts
const span = grid.getCellSpan(row, column);
// → { rowSpan: number, colSpan: number, hidden: boolean }
```

어댑터는 `hidden`이면 `td`를 렌더링하지 않고, `rowSpan`/`colSpan`이 1보다
크면 해당 속성을 부여한다 — 4개 어댑터 모두 동일하게 처리된다.

## 예시 데이터

```ts
const data = [
  { team: "개발", name: "Hana",  role: "FE" },
  { team: "개발", name: "Daeho", role: "BE" },
  { team: "개발", name: "Bora",  role: "BE" },
  { team: "영업", name: "Felix", role: "Sales" },
];
// team 컬럼 merge:"row" → "개발"은 rowSpan=3으로 한 셀, "영업"은 1
```
