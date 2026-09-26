# CSV 보내기 / 가져오기 / 템플릿

현재 표시 상태(필터·검색·정렬·페이징·그룹 펼침이 반영된 데이터)를 CSV로
보내고, CSV 파일을 행으로 가져온다. 그리드 컬럼에 맞는 템플릿 다운로드도
제공한다.

## 코어 API

```ts
const csv = grid.exportToCsv({
  filename: "users.csv",        // 다운로드 파일명 (기본값 "grid.csv")
  visibleColumnsOnly: true,     // 숨긴 컬럼 제외 (기본값)
  selectedRowsOnly: false,      // true면 선택된 행만 (기본값)
});
```

- 반환값: `\uFEFF`(UTF-8 BOM)가 선행된 CSV 문자열 — 엑셀에서 한글이
  깨지지 않는다.
- 브라우저 환경에서는 파일 다운로드까지 트리거되고, 비-DOM 환경
  (테스트/SSR)에서는 문자열만 반환한다.
- 서버 사이드 모드에서는 로드된 행만 출력된다.

## 직렬화 규칙

- 셀 값은 `formatter` 적용 표시 텍스트 기준.
- `,` `"` `\n` `\r`을 포함한 셀은 큰따옴표로 감싸고 내부 `"`는 `""`로
  이스케이프 (RFC 4180, CRLF 행 구분).
- 그룹화 중이면 펼쳐진 리프 행만 표시 순서대로 출력 (그룹 헤더 제외).

## 순수 함수

개별 export로 직접 조합 가능하다:

```ts
import {
  escapeCsvCell, buildCsv, downloadCsv, CSV_BOM,
  parseCsv, coerceCsvCell,
} from "@moda-grid/core";
```

## CSV 가져오기 (Import)

`grid.importCsv(csvText, options)`가 CSV 문자열을 파싱해 행을 만든다:

```ts
const result = grid.importCsv(text, {
  hasHeader: true,   // 첫 줄을 컬럼 매칭에 사용 (기본값)
  replace: false,    // false=뒤에 추가, true=데이터 교체 (기본값)
});
// result: { added: 2, skipped: 1, errors: ["3행: 나이 — 숫자가 아닙니다 (\"abc\")"] }
```

- **헤더 매칭** (`hasHeader: true`, 기본): 첫 줄을 `header ?? field` 또는
  `field` 이름으로 컬럼에 매칭한다. CSV의 컬럼 순서가 달라도 안전하고,
  매칭되지 않는 열은 무시된다. 숨긴 컬럼도 이름으로 매칭된다.
- **위치 매칭** (`hasHeader: false`): 표시 컬럼 순서대로 셀을 배정한다.
- **형변환**: `cellEditor`/`filterType`이 `'number'`인 컬럼은 Number로
  변환 (빈 문자열 → null, 변환 불가 → 행 건너뜀 + 에러).
- **`valueSetter`**가 있으면 그 경로로 쓰고, **`validate`** 실패 행은
  건너뛰고 `errors`에 행 번호와 사유를 기록한다.
- **`required`** 컬럼도 검사한다 — CSV에 값이 비었거나 해당 컬럼 자체가
  헤더에 없어 비어 있으면 필수 에러로 행을 건너뛴다.
- 파서는 RFC 4180 대응 — 따옴표 필드 안의 쉼표/줄바꿈, `""` 이스케이프,
  CRLF/LF, 선행 BOM을 처리한다. `exportToCsv` 출력은 그대로 재가져올 수
  있다 (round-trip).

## CSV 템플릿

`grid.getCsvTemplate(options)`는 **그리드의 컬럼 정의에 맞는 헤더 행만**
담긴 CSV를 반환하고 다운로드한다:

```ts
grid.getCsvTemplate({
  filename: "users-template.csv",  // 기본값 "template.csv"
  visibleColumnsOnly: false,       // 숨긴 컬럼 제외 (기본값 false — 전부 포함)
});
```

export와 같은 헤더 규칙(`header ?? field`)을 쓰므로 사용자가 채운 파일을
`importCsv`에 바로 넣을 수 있다. "템플릿 다운로드 → 엑셀에서 작성 →
가져오기" 워크플로우에 쓴다.

## 버튼 예시

```tsx
// React
<button onClick={() => grid.exportToCsv({ filename: "users.csv" })}>CSV보내기</button>
<button onClick={() => fileRef.current?.click()}>CSV 가져오기</button>
<input
  ref={fileRef}
  type="file"
  accept=".csv,text/csv"
  hidden
  onChange={async (e) => {
    const file = e.target.files?.[0];
    if (file) grid.importCsv(await file.text());
    e.target.value = "";
  }}
/>
<button onClick={() => grid.getCsvTemplate()}>CSV 템플릿</button>
```
```vue
<!-- Vue -->
<button @click="grid.exportToCsv({ filename: 'users.csv' })">CSV보내기</button>
<button @click="csvFile?.click()">CSV 가져오기</button>
<input ref="csvFile" type="file" accept=".csv,text/csv" hidden @change="onCsvFile" />
<button @click="grid.getCsvTemplate()">CSV 템플릿</button>
```
```svelte
<!-- Svelte -->
<button onclick={() => grid.exportToCsv({ filename: "users.csv" })}>CSV보내기</button>
<button onclick={() => csvFile?.click()}>CSV 가져오기</button>
<input bind:this={csvFile} type="file" accept=".csv,text/csv" hidden onchange={onCsvFile} />
<button onclick={() => grid.getCsvTemplate()}>CSV 템플릿</button>
```

전체 구현은 `apps/dev-*` 툴바의 CSV 가져오기/템플릿 버튼을 참고한다.
