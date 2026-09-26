# moda-grid 사용자 가이드

멀티 프레임워크 데이터 그리드 라이브러리. 헤드리스 코어(`@moda-grid/core`)
위에 React / Vue 3 / Vue 2.7 / Svelte 5 어댑터와 CDN(순수 JS) 번들을 제공한다.

이 디렉토리는 **사용자 가이드** — 개발자와 AI가 기능을 찾아 바로 적용할 수
있도록 플랫폼별·기능별로 구성되어 있다. 소스 레포의 `docs/`가 GitHub Action에
의해 공개 레포(`moda-grid-releases`)로 자동 배포된다.

> 개발자 문서(아키텍처·빌드·벤치마크)는 상위 [docs/README.md](../README.md) 참고.

## 목차

### 시작하기

| 문서 | 내용 |
| ---- | ---- |
| [getting-started.md](./getting-started.md) | 설치, Hello Grid, 핵심 개념(스냅샷·파이프라인), 패키지 선택 |

### 플랫폼별 가이드

| 플랫폼 | 패키지 | 문서 |
| ------ | ------ | ---- |
| 순수 JS (CDN) | `moda-grid.js` (IIFE) | [platforms/vanilla-js.md](./platforms/vanilla-js.md) |
| React | `@moda-grid/react` | [platforms/react.md](./platforms/react.md) |
| Vue 3 | `@moda-grid/vue` | [platforms/vue3.md](./platforms/vue3.md) |
| Vue 2.7 | `@moda-grid/vue2` | [platforms/vue2.md](./platforms/vue2.md) |
| Svelte 5 | `@moda-grid/svelte` | [platforms/svelte.md](./platforms/svelte.md) |

### 기능별 가이드

모든 기능은 코어(`GridCore`)가 담당하고 어댑터는 동일하게 렌더링하므로,
기능 문서의 코어 API는 플랫폼과 무관하게 동일하다.

| 카테고리 | 기능 | 문서 |
| -------- | ---- | ---- |
| 데이터 | 정렬 / 다중 정렬 | [features/sorting.md](./features/sorting.md) |
| 데이터 | 컬럼 필터 / 전역 검색 / Set 필터 | [features/filtering.md](./features/filtering.md) |
| 데이터 | 페이징 | [features/paging.md](./features/paging.md) |
| 데이터 | 행 그룹화 + 집계 | [features/grouping.md](./features/grouping.md) |
| 데이터 | 트리 데이터 (계층) | [features/tree-data.md](./features/tree-data.md) |
| 데이터 | 서버 사이드 / 무한 스크롤 | [features/server-side.md](./features/server-side.md) |
| 컬럼 | 리사이즈·재배치·고정·숨김·자동 너비·그룹 헤더·컬럼 관리 | [features/columns.md](./features/columns.md) |
| 선택 | 행/셀/범위 선택 + 체크박스 + 상태바 집계 | [features/selection.md](./features/selection.md) |
| 편집 | 인라인 편집 + 검증 + 커스텀 에디터 + Undo/Redo | [features/editing.md](./features/editing.md) |
| 편집 | 클립보드 복사/붙여넣기 (TSV) | [features/clipboard.md](./features/clipboard.md) |
| 출력 | CSV 보내기/가져오기/템플릿 | [features/csv-export.md](./features/csv-export.md) |
| 행 | 행 상태 추적 (I/U/D) + 변경분 수집 | [features/row-state.md](./features/row-state.md) |
| 행 | 행 드래그앤드롭 재정렬 | [features/row-drag.md](./features/row-drag.md) |
| 행 | 고정 행 + 전체 총계 | [features/pinned-rows.md](./features/pinned-rows.md) |
| 성능 | 가상 스크롤 | [features/virtual-scroll.md](./features/virtual-scroll.md) |
| 상태 | 상태 저장/복원 (localStorage) | [features/state-persistence.md](./features/state-persistence.md) |
| 스타일 | 테마 / 다크 모드 / 커스텀 클래스 | [features/theming.md](./features/theming.md) |

### 기여

| 문서 | 내용 |
| ---- | ---- |
| [extending.md](./extending.md) | 새 기능 개발 시 가이드 문서 추가 규칙 (개발자·AI 공용) |

## 30초 요약

```ts
// 어떤 플랫폼이든 동일한 코어 API
const grid = new GridCore({ columns, data });
grid.toggleSort("age");              // 정렬
grid.setFilter("role", "admin");     // 필터
grid.setSearch("hana");              // 전역 검색
grid.setPage(0, 20);                 // 페이징
grid.exportToCsv({ filename: "u.csv" });
```

- **스냅샷 패턴**: `grid.subscribe(cb)` + `grid.getSnapshot()` — 모든 상태는
  불변 스냅샷으로 읽고, 변경은 `grid.*()` 액션으로만 수행한다.
- **DataGrid props**: `columns`/`data`(또는 `grid`), `height`+`rowHeight`
  (가상 스크롤), `selectionMode`, `columnController`, `rowCheckboxes` 등은
  4개 어댑터 모두 동일하다.
