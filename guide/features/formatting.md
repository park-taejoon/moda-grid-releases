# 내장 셀 포맷터 (ColumnDef.format)

숫자·통화·퍼센트·날짜를 선언형 포맷으로 표시한다. `formatter`(커스텀 함수)
보다 간단하고, IBSheet의 내장 포맷과 같은 개념이다.

## 적용 우선순위

```
column.formatter (커스텀 함수) > column.format (내장 포맷) > String(value)
```

`getCellText` 경로로 적용되므로 셀 표시·CSV/xlsx보내기·복사에 동일하게 반영된다.

## 포맷 종류

```ts
const columns = [
  // number — 천단위 콤마, 소수점, 접두/접미사
  { field: "price", format: { type: "number" } },
  { field: "qty",   format: { type: "number", decimals: 2, suffix: "개" } },

  // currency — number + 통화 기호 (기본 "₩")
  { field: "amount", format: { type: "currency" } },            // ₩1,234
  { field: "usd",    format: { type: "currency", symbol: "$", decimals: 2 } },

  // percent — 0.15 → "15%" (1을 100%로 봄)
  { field: "ratio", format: { type: "percent", decimals: 1 } }, // 15.0%

  // date — 토큰 패턴. 값은 Date / timestamp(ms) / 파싱 가능한 문자열
  { field: "created", format: { type: "date" } },                       // YYYY-MM-DD
  { field: "at",      format: { type: "date", pattern: "YYYY.MM.DD HH:mm" } },
];
```

| 타입 | 옵션 | 기본값 | 예 |
| ---- | ---- | ------ | -- |
| `number` | `decimals`, `thousandsSeparator`, `prefix`, `suffix` | 소수점 입력 그대로, 콤마 true | `1,234.56` |
| `currency` | `symbol`, `decimals` | `₩`, 0 | `₩1,234` |
| `percent` | `decimals` | 0 | `15%` |
| `date` | `pattern` | `YYYY-MM-DD` | `2026.09.26 14:30` |

날짜 패턴 토큰: `YYYY` `YY` `MM` `DD` `HH` `mm` `ss`

## 변환 불가 값

`null`/`undefined`는 빈 문자열, 파싱 실패 값은 원본 문자열로 표시된다 —
포맷 실패로 셀이 비거나 깨지지 않는다.

## 직접 호출

```ts
import { applyCellFormat, formatCellText } from "@moda-grid/core";

applyCellFormat(1234.5, { type: "number", decimals: 1 }); // "1,234.5"
```
