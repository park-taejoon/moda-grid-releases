# Excel (xlsx) 보내기 / 가져오기

현재 표시 데이터를 `.xlsx`로보내고, xlsx 파일을 행으로 가져온다.
SheetJS(`xlsx` 패키지)를 **첫 호출 시 지연 로드(dynamic import)**하므로
기능을 쓰지 않는 화면의 초기 번들에는 영향이 없다.

## xlsx보내기

```ts
const buf = await grid.exportToXlsx({
  filename: "users.xlsx",        // 다운로드 파일명 (기본값 "grid.xlsx")
  sheetName: "사용자",            // 시트명 (기본값 "Sheet1")
  visibleColumnsOnly: true,      // 숨긴 컬럼 제외 (기본값)
  selectedRowsOnly: false,       // true면 선택 행만 (기본값)
});
```

- 반환값: `ArrayBuffer` — 브라우저에서는 다운로드도 트리거되고,
  비-DOM 환경(테스트/SSR)에서는 버퍼만 반환한다.
- `cellEditor`/`filterType`이 `'number'`인 컬럼은 **숫자 셀**로 써서
  Excel에서 바로 합산/계산할 수 있다. 그 외 컬럼은 `formatter`/`format`
  적용 표시 텍스트.
- 필터·정렬·그룹 펼침 등 현재 표시 상태가 그대로 반영된다 (CSV와 동일 규칙).
- 서버 사이드 모드에서는 로드된 행만 출력된다.

## 스타일 반영 보내기 (`styled: true`)

`styled: true`를 주면 SheetJS 대신 **exceljs**로 생성해 실제 xlsx 서식을
포함한다 (첫 호출 시 exceljs가 지연 로드된다):

```ts
await grid.exportToXlsx({ styled: true, filename: "report.xlsx" });
```

반영되는 서식:

- **헤더 행** — 굵게 + 연한 회색 배경 + 하단 테두리
- **컬럼 너비** — `col.width`를 Excel 열 너비로 환산
- **표시 형식** — `format`이 있는 컬럼은 원시 값 + Excel `numFmt`로 쓴다.
  Excel에서 값이 계산 가능하며 표시만 포맷된다:
  - `number` → `#,##0` / `#,##0.00` (decimals·천단위·prefix·suffix 반영)
  - `currency` → `"₩"#,##0`, `percent` → `0.00%`
  - `date` → 패턴 토큰을 Excel 형식으로 변환 (`YYYY-MM-DD` → `yyyy-mm-dd`)
- **테두리** — 본문 전 셀 thin
- **행 병합** — `merge: "row"|"both"` 컬럼은 같은 표시 값의 연속 행이
  실제 xlsx 병합 셀이 된다

파일 크기가 민감하거나 서식이 필요 없으면 기본(styled:false) 경로를
사용한다 — SheetJS가 더 가볍다.

## xlsx 가져오기

`importCsv`와 **동일한 매칭·검증·행 상태 규칙**을 공유한다:

```ts
const result = await grid.importXlsx(fileOrBuffer, {
  hasHeader: true,   // 첫 행을 header ?? field / field로 매칭 (기본값)
  replace: false,    // true면 데이터 교체
  sheet: "Sheet2",   // 또는 인덱스 (기본 첫 시트)
});
// result: { added, skipped, errors } — importCsv와 동일 형태
```

- `File`/`Blob`/`ArrayBuffer`/`Uint8Array` 입력을 받는다.
- 숫자·불리언 셀은 타입을 유지하고, 날짜 셀은 ISO 문자열로 변환된다.
- `valueSetter`/`validate`/`required` 검증과 `I`(Inserted) 행 마킹도
  CSV와 같다 — `exportToXlsx` 출력은 그대로 재가져올 수 있다 (round-trip).
- 시트가 없으면 `errors`에 사유를 담아 반환한다 (throw 아님).

## 버튼 예시

```tsx
// React
<button onClick={() => grid.exportToXlsx({ filename: "users.xlsx" })}>
  Excel보내기
</button>
<input
  ref={fileRef}
  type="file"
  accept=".xlsx,.xls"
  hidden
  onChange={async (e) => {
    const file = e.target.files?.[0];
    if (file) await grid.importXlsx(file);
    e.target.value = "";
  }}
/>
```

## 참고

- 번들 고려사항: `xlsx`는 ~1MB 규모라 dynamic import 청크로 분리된다.
  CDN 단일 파일 번들에는 포함될 수 있다.
- 스타일/서식 보존 보내기는 지원하지 않는다 — 데이터만 직렬화된다.
- CSV가 필요하면 [csv-export.md](./csv-export.md) 참고.
