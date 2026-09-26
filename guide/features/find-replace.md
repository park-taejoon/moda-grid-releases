# 찾기 / 바꾸기

그리드 데이터를 텍스트로 검색·일괄 치환하는 코어 API — IBSheet의
`findText`/`replaceText`에 해당한다. 어댑터 종류와 무관하게
`GridCore` 메서드로 동작하므로 헤드리스로도 쓸 수 있다.

## `findCells`

표시 텍스트(`getCellText` — 포맷터 적용 결과) 기준으로 검색하고,
매치된 셀의 위치를 반환한다.

```ts
const matches = grid.findCells("긴급");
// FindMatch[] = [{ rowIndex, columnIndex, row, column }, ...]

matches.forEach((m) => {
  console.log(m.rowIndex, m.column.field, m.row);
});

// 옵션
grid.findCells("ABC", { caseSensitive: true }); // 대소문자 구분
grid.findCells("정확히", { wholeCell: true });  // 셀 전체 일치만
```

- `rowIndex`/`columnIndex`는 `visibleData`/`visibleColumns` 기준 인덱스
  — `setActiveCell(m.rowIndex, m.columnIndex)`로 바로 이동 가능하다.
- 필터/검색/정렬이 적용된 **현재 표시 상태**를 검색한다.
- 빈 검색어는 빈 배열을 반환한다.

## `replaceAll`

매치된 모든 셀에서 `find`를 `replace`로 치환해 저장한다. 반환값은
실제로 변경된 셀 수.

```ts
const changed = grid.replaceAll("대기", "진행");
grid.replaceAll("abc", "x", { caseSensitive: true });
grid.replaceAll("구버전", "v2", { wholeCell: true });
```

### 동작 규칙

- **편집과 동일한 쓰기 경로**를 거친다:
  - `editable: false` 컬럼과 `formula` 컬럼은 건너뛴다.
  - `valueSetter` 컬럼은 세터를 통해 저장된다.
  - `column.validate`/`required` 검증 실패 셀은 건너뛴다.
  - `cellEditor: "number"` 등 타입 컬럼은 치환 문자열을 저장 타입으로
    변환한다 (`"99"` → `99`).
- 변경된 셀은 `afterEdit` 이벤트를 발생시키고, 소속 행은 `U`로 마킹된다.
- 실제 값이 변하지 않는 치환(동일 결과)은 이력에 남지 않고 `U`도 찍지
  않는다.
- 치환 전체가 **Undo 1단위**로 기록된다 — `grid.undo()` 한 번에 모두
  되돌아간다.

### `findCells`와의 차이

`findCells`는 **표시 텍스트**(포맷 적용)를 검색하지만, `replaceAll`의
치환은 **원시 값의 문자열 표현**에 적용된다. 포맷터(`format`)가 붙은
컬럼(예: `1,234`)은 표시 텍스트로 매치되더라도 치환은 원시값(`1234`)
기준으로 이뤄진다 — 표시 포맷만으로 매치된 셀은 치환되지 않을 수 있다.

## 검색 후 이동 예시

```ts
const hits = grid.findCells(query);
if (hits[0]) {
  grid.setActiveCell(hits[0].rowIndex, hits[0].columnIndex);
  grid.scrollToRow?.(hits[0].rowIndex); // 가상 스크롤 시
}
```
