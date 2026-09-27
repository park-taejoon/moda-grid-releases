# 찾기 / 바꾸기

그리드 데이터를 텍스트로 검색·일괄 치환하는 코어 API — 타사 그리드의
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

## 검색 후 이동 — `findNext`

`findCells` 결과를 직접 순회할 필요 없이, 활성 셀 다음 위치부터 첫 매치로
이동시키는 편의 API다. 목록 끝에 도달하면 처음으로 순환한다 — 엑셀/타사 그리드
"다음 찾기"와 같은 UX:

```ts
const m = grid.findNext("긴급");      // 활성 셀 이후 첫 매치로 이동
if (m) console.log(m.rowIndex, m.column.field);
grid.findNext("긴급");                // 다시 호출 → 다음 매치 (순환)
grid.findNext("긴급", { backward: true }); // 이전 매치 방향으로 순환
```

- 매치가 없으면 `null`을 반환하고 활성 셀은 바뀌지 않는다.
- 이동 순서는 행 우선(row-major) — 같은 행이면 오른쪽 컬럼부터.
- `backward: true`는 활성 셀 이전으로 거슬러 올라가며, 처음에서 끝으로 순환한다.

### 내장 찾기 바 — `findBox`

`searchBox`(데이터 필터)와 별개로, 셀 텍스트를 검색해 매치로 이동하는
내장 창을 켤 수 있다 — 5개 렌더러 모두 지원:

```tsx
// React/Vue3/Vue2/Svelte — prop
<DataGrid columns={cols} data={rows} findBox />
```

```ts
// vanilla
mountGrid(el, { columns, data, findBox: true });
```

- `Enter` = 다음 매치, `Shift+Enter` = 이전 매치, 버튼 클릭도 동일.
- 검색 옵션 체크박스 — **대소문자 구분**(`caseSensitive`)과
  **셀 전체 일치**(`wholeCell`)를 지원한다 (타사 그리드 찾기 옵션 대응).
  라벨은 `locale.findMatchCase`/`findWholeCell`로 다국어 대응한다.
- 그리드에 포커스가 있을 때 **`Ctrl+F`/`Cmd+F`로 찾기 입력에 포커스**
  (기존 검색어 전체 선택). `findBox`가 꺼져 있으면 브라우저 기본 검색에 양보한다.
- 매치 총 건수를 표시하고, 찾은 셀로 `scrollIntoView`한다.
- **입력 즉시 매치 셀에 `mg-find-match` 배경이 칠해진다** — 입력을
  지우면 해제. 색상은 `--grid-find-match-bg`/`--grid-find-current-bg`
  변수로 테마 대응한다.
- 내부적으로 `findCells`/`findNext`를 사용한다 — 같은 동작을 직접 구현할
  수도 있다.

### 매치 하이라이트 API — `setFindQuery` / `isFindMatch`

findBox 없이 하이라이트만 제어할 수 있다:

```ts
grid.setFindQuery("서울");        // 매치 셀에 mg-find-match 클래스
grid.setFindQuery("서울", { wholeCell: true });
grid.setFindQuery(null);          // 해제
grid.isFindMatch(row, "region");  // boolean — 커스텀 렌더링용
snapshot.findQuery;               // 현재 하이라이트 검색어 | null
```

매치 판정은 `getCellText`(표시 텍스트) 기준이며 `"행ID|필드"` 캐시로
계산된다 — 정렬/페이지 이동/그룹 토글로 표시 인덱스가 바뀌어도 정확하고,
필터로 빠진 행은 매치하지 않는다.

## 인덱스 기반 직접 읽기/쓰기 — `getCellValueAt` / `setCellValue`

타사 그리드 `getValue(r, c)`/`setValue(r, c, v)`에 해당하는 인덱스 기반 API.
인덱스는 `visibleData`/`visibleColumns` 기준이다:

```ts
grid.getCellValueAt(0, 1);            // 읽기
grid.setCellValue(0, 1, 99);          // 쓰기 — 편집 커밋과 동일 경로
```

`setCellValue`는 인라인 편집과 같은 규칙을 거친다:

- `editable: false`·수식 컬럼은 `false`를 반환하고 쓰지 않는다
- `column.validate`/`required` 실패 시 `false` — 값은 바뀌지 않는다
- `valueSetter` 컬럼은 세터를 통해 저장된다
- 변경 시 행 상태(U)·Undo 이력·`afterEdit` 이벤트에 기록된다
- 값이 같으면 이력을 남기지 않고 `true`를 반환한다 (팬텀 U 방지)

## 행 ID 기반 읽기/쓰기 — `getRowById` / `updateRow`

타사 그리드 `getRowData`/`setRowData`에 해당하는 ID 기반 API — 인덱스가
정렬·필터로 바뀌어도 안전하다:

```ts
grid.getRowById("42");                    // TData | undefined (숨김 행 포함)
grid.updateRow("42", { name: "새 이름", age: 30 }); // 부분 업데이트 → 적용 셀 수
```

`updateRow` 규칙:

- 컬럼이 있는 필드는 `setCellValue`와 같은 규칙 — `editable:false`·수식·
  검증 실패 필드는 건너뛰고 `valueSetter`를 경유한다
- 컬럼에 없는 필드는 직접 할당한다 (검증 없음 — 메타/외래키 필드용)
- 실제로 바뀐 셀만 모아 **한 Undo 단위**로 기록, 행 상태 U 마킹,
  `dataChange`(source `"api"`) 발행 — 필드별 `afterEdit`은 발행하지 않는다
- 변경이 없으면 0을 반환하고 이력을 남기지 않는다
