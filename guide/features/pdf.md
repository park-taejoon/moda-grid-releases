# PDF 보내기

현재 표시 데이터를 PDF 표로 저장한다 — IBSheet PDF 다운로드에 해당한다.
**jspdf + jspdf-autotable**을 첫 호출 시 지연 로드(dynamic import)한다.

## 사용법

```ts
await grid.exportToPdf({
  filename: "report.pdf",     // 기본값 "grid.pdf"
  title: "2026 매출 보고서",   // 페이지 상단 제목 (선택)
  orientation: "landscape",   // 기본값 "portrait"
  fontSize: 9,                // 본문 글자 크기 pt (기본값 9)
  visibleColumnsOnly: true,   // 숨긴 컬럼 제외 (기본값)
  selectedRowsOnly: false,    // true면 선택 행만
});
```

- 필터·정렬·그룹 펼침 등 **현재 표시 상태**가 그대로 반영된다
  (CSV/xlsx와 동일한 행·컬럼 해석 규칙).
- 셀 값은 `formatter`/`format`이 적용된 표시 텍스트로 출력된다.
- 서버 사이드 모드에서는 로드된 행만 출력된다.

## 한글/비Latin 문자 — `fontUrl`

jsPDF의 기본 폰트(Helvetica)는 한글을 지원하지 않는다. 한국어 데이터가
있으면 TTF 폰트를 임베드해야 한다:

```ts
await grid.exportToPdf({
  filename: "보고서.pdf",
  title: "매출",
  // Noto Sans KR 등 TTF 폰트 URL — 첫 호출 시 fetch해 문서에 임베드
  fontUrl: "https://example.com/fonts/NotoSansKR-Regular.ttf",
});
```

- 같은 PDF 문서 생성 호출마다 폰트를 fetch하므로, 반복 보내기가 많으면
  폰트를 로컬/캐시 가능한 경로에 두는 것이 좋다.
- 영문/숫자만 있는 데이터는 `fontUrl` 없이 동작한다.

## 대안 — 브라우저 인쇄

`window.print()`로도 인쇄할 수 있다. `styles.css`에 `@media print` 규칙이
포함되어 있어 컨텍스트 메뉴·컬럼 관리·로딩 표시 등 팝업 UI는 인쇄물에서
자동으로 제외되고, 행이 페이지 넘김에서 잘리지 않는다.
