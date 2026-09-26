# 클립보드 — Copy & Paste (TSV)

엑셀 호환 TSV 복사/붙여넣기. 어댑터가 브라우저 클립보드 이벤트를 코어 API에
연결한다.

## 사용자 조작 (DataGrid 기본 동작)

| 단축키 | 동작 |
| ------ | ---- |
| `Ctrl/Cmd + C` | 선택 범위(없으면 활성 셀)를 TSV로 복사 |
| `Ctrl/Cmd + Shift + C` | 헤더 행 포함 복사 |
| `Ctrl/Cmd + V` | 활성 셀부터 오른쪽/아래로 TSV 붙여넣기 |

입력 요소 내부에서는 브라우저 기본 동작이 유지된다.

## 코어 API

```ts
grid.getSelectionTsv();                          // 선택 범위 → TSV 문자열 (없으면 null)
grid.getSelectionTsv({ includeHeaders: true });  // 첫 줄에 헤더 행 포함
grid.pasteTsv(tsv);                              // 활성 셀부터 순차 쓰기 → PasteResult
grid.pasteTsv(tsv, { rowIndex: 0, columnIndex: 0 }); // 시작 위치 지정
```

`PasteResult` = `{ applied, skipped, errors }`.

## 붙여넣기 규칙

- `editable: false` 컬럼과 범위 밖 셀은 건너뜀 (`skipped`에 기록).
- `cellEditor: 'number'` 컬럼은 숫자로 변환 (변환 불가 시 에러 기록).
- `column.validate` 실패 시 건너뜀 + `errors`에 위치·메시지 기록.
- 쓰기는 `valueSetter` 또는 `row[field] = value` — 편집 커밋과 동일 경로.
- 적용된 붙여넣기는 **1회 `notify`** + Undo 이력에 1개 단위로 기록.

## 붙여넣기 행 확장 (`pasteExtend`)

`GridOptions.pasteExtend: true`면 붙여넣은 줄 수가 표시 데이터를 넘을 때
부족한 만큼 **빈 행을 자동 추가**하고 붙여넣는다 (IBSheet `EditExtend` 대응).

```ts
new GridCore({ columns, data, pasteExtend: true });
```

- 추가된 행은 **입력(I) 상태**로 마킹 — `getChanges().inserted`에 잡혀
  저장 대상이 된다.
- 기존 행 수정분과 새 행의 셀 쓰기는 하나의 붙여넣기로 Undo 1단위다.
- 서버 모드(`serverSide`)와 페이징(`pageSize > 0`)에서는 확장하지 않는다
  — 표시 범위 밖 행에 쓸 수 없기 때문. 넘어가는 줄은 `skipped`로 집계.

## 복사 규칙

- 숨김 컬럼 제외, 셀 내부 탭/줄바꿈은 공백으로 치환.
- 표시 상태(필터·정렬·페이징 결과) 기준으로 복사된다.

## 어댑터 연결 방식 (커스텀 UI용)

```ts
// 어댑터가 내부적으로 하는 일
document.addEventListener("copy", ...);   // navigator.clipboard.writeText(grid.getSelectionTsv())
// paste: const text = await navigator.clipboard.readText(); grid.pasteTsv(text);
```

비-어댑터 환경(CDN 직접 렌더링)에서는 위 패턴을 그대로 구현하면 된다.
