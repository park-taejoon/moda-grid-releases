# CSV보내기 (CSV Export)

현재 표시 상태(필터·검색·정렬·페이징·그룹 펼침이 반영된 데이터)를 CSV로
보낸다.

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
import { escapeCsvCell, buildCsv, downloadCsv, CSV_BOM } from "@moda-grid/core";
```

## 버튼 예시

```tsx
// React
<button onClick={() => grid.exportToCsv({ filename: "users.csv" })}>CSV보내기</button>
```
```vue
<!-- Vue -->
<button @click="grid.exportToCsv({ filename: 'users.csv' })">CSV보내기</button>
```
```svelte
<!-- Svelte -->
<button onclick={() => grid.exportToCsv({ filename: "users.csv" })}>CSV보내기</button>
```
