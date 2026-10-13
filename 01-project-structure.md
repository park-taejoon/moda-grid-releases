# 프로젝트 구조

```
moda-grid/
├── package.json            # 루트 스크립트, 공유 typescript·eslint·prettier
├── pnpm-workspace.yaml     # packages/*, apps/* 워크스페이스 선언
├── tsconfig.base.json      # 모든 패키지가 상속하는 TS 공통 설정
├── eslint.config.mjs       # 플랫 설정 — typescript-eslint + react-hooks
├── packages/
│   ├── core/               # @moda-grid/core   — 순수 TS 핵심 로직
│   │   └── src/
│   │       ├── types/      #   도메인별 타입 선언 (columns/cells/data/
│   │       │               #   dnd/events/menu/options/server/state)
│   │       ├── types.ts    #   위 도메인 모듈의 export * 배럴 (공개 타입)
│   │       ├── grid.ts     #   GridCore facade — 공개 API 위임만 수행
│   │       ├── gridFull.ts #   features 미지정 시 allFeatures 주입 (호환)
│   │       ├── slim.ts     #   슬림 엔트리 — bare 커널 + 기능 팩토리
│   │       ├── kernel/     #   GridHost/GridFeature 규약, inert 스텁,
│   │       │               #   measure/values/rowFlip/surface 헬퍼
│   │       ├── features/   #   기능 컨트롤러 (columns/selection/editing/
│   │       │               #   clipboard/sorting/filtering/grouping/
│   │       │               #   pivot/serverSide/rowDrag/… 26종)
│   │       ├── mount.ts    #   mountGrid — vanilla DOM 렌더러 엔트리
│   │       ├── mount/      #   vanilla 렌더러 모듈 (ctx/dom/findBar/
│   │       │               #   touchDnd/header/panels/body/bodyRows/
│   │       │               #   cellEditor)
│   │       ├── conformance.ts # 5렌더러 공유 DOM 계약 스펙
│   │       ├── cellMerge.ts / aggregates.ts / cumulative.ts
│   │       │               #   grid.ts에서 추출한 순수 계산
│   │       ├── displayRows.ts # DisplayRow 합집합 판별·추출 헬퍼
│   │       │               #   (5개 렌더러의 행 해석 공유)
│   │       ├── adapterDom.ts # 5개 렌더러 공용 DOM/제스처/키보드 계약
│   │       │               #   (pin·드롭 수학·리사이즈·fill·keydown)
│   │       ├── virtualScroll.ts / virtualColumns.ts / filtering.ts /
│   │       │   grouping.ts / pivot.ts / csv.ts / xlsx.ts / pdf.ts /
│   │       │   formula.ts / format.ts / theme.ts / touch.ts /
│   │       │   menuDefaults.ts / clipboardIO.ts / locale.ts
│   │       ├── styles.css  #   공통 .mg-* 스타일 (단일 소스)
│   │       ├── *.test.ts   #   vitest (기능별 + inert 충실도 + mount)
│   │       └── index.ts
│   ├── react/              # @moda-grid/react  — React 어댑터
│   │   └── src/
│   │       ├── useGridCore.ts  # useGridCore / useGridSnapshot 훅
│   │       ├── DataGrid.tsx    # 테이블 + 가상 스크롤 + renderCell 컴포넌트
│   │       ├── styles.css      # @import "@moda-grid/core/styles.css"
│   │       └── index.ts
│   ├── vue/                # @moda-grid/vue    — Vue 3 어댑터
│   │   └── src/
│   │       ├── useGrid.ts      # useGrid / useGridState 컴포저블 (shallowRef)
│   │       ├── DataGrid.vue    # <script setup> SFC + 스코프드 슬롯 + 가상 스크롤
│   │       └── styles.css
│   ├── vue2/               # @moda-grid/vue2   — Vue 2.7 어댑터 (내장 Composition API)
│   │   └── src/
│   │       ├── useGrid.ts      # useGrid / useGridState 컴포저블
│   │       ├── DataGrid.vue    # Vue 2.7 <script setup> SFC (런타임 props)
│   │       └── styles.css
│   └── svelte/             # @moda-grid/svelte — Svelte 어댑터
│       ├── svelte.config.js    # svelte-package / vitePreprocess 설정
│       └── src/lib/
│           ├── gridStore.ts    # createGridStore / toGridStore (readable 스토어)
│           ├── DataGrid.svelte # runes 컴포넌트 + {#snippet cell} + 가상 스크롤
│           ├── types.ts        # CellContext / DataGridProps
│           └── styles.css
├── apps/
│   ├── dev-react/          # Vite + React 데모 (포트 5173)
│   ├── dev-vue/            # Vite + Vue 3 데모 (포트 5174)
│   ├── dev-svelte/         # Vite + Svelte 5 데모 (포트 5175)
│   ├── dev-vue2/           # Vite + Vue 2.7 데모 (포트 5176)
│   ├── dev-vanilla/        # mountGrid 데모 (포트 5177)
│   ├── demo-shared/        # 데모 공통 데이터·컬럼·체크리스트·셀렉트 옵션
│   └── bench/              # grid.bench.ts 실행 앱
├── e2e/                    # Playwright 스펙 (역할별 14개 + helpers.ts)
├── scripts/                # dev/demo/deploy/release/pre-push 스크립트
└── docs/                   # 사용자 문서 (guide/ 하위 가이드 포함)
```

## 패키지 의존 관계

```
@moda-grid/core  (의존성 없음)
@moda-grid/react    → @moda-grid/core (workspace:*), peer: react, react-dom
@moda-grid/vue      → @moda-grid/core (workspace:*), peer: vue >= 3.3
@moda-grid/svelte   → @moda-grid/core (workspace:*), peer: svelte ^5
@moda-grid/vue2     → @moda-grid/core (workspace:*), peer: vue ^2.7

demo-shared → @moda-grid/core (workspace:*)
dev-react   → @moda-grid/react (workspace:*), @moda-grid/demo-shared
dev-vue     → @moda-grid/vue   (workspace:*), @moda-grid/demo-shared
dev-svelte  → @moda-grid/svelte(workspace:*), @moda-grid/demo-shared
dev-vue2    → @moda-grid/vue2  (workspace:*), @moda-grid/demo-shared
dev-vanilla → @moda-grid/core  (workspace:*), @moda-grid/demo-shared
```

`workspace:*` 프로토콜로 선언되어 pnpm이 항상 로컬 패키지를 링크하고,
`pnpm -r build` 실행 시 의존 그래프 토폴로지 순서(core → 어댑터)로 빌드한다.

## 소스 직접 참조 (source-first exports)

내부 패키지의 `exports`는 빌드 산출물이 아니라 **소스 파일**을 가리킨다.

```jsonc
// packages/core/package.json
"exports": {
  ".": { "types": "./src/index.ts", "default": "./src/index.ts" },
  "./styles.css": "./src/styles.css"
}
```

효과:

- dev 앱의 Vite가 패키지 소스를 직접 변환하므로 **코어 수정이 HMR로 즉시 반영**된다.
- 각 패키지의 `pnpm build`는 배포용 `dist/` 산출물을 생성한다
  (core/react/vue는 `tsc`, svelte는 `svelte-package`).
- npm 배포 시 `publishConfig.exports`가 `dist/`로 자동 치환된다 —
  개발은 소스 직접 참조, 배포는 빌드 산출물이라 두 세계가 공존한다.

## tsconfig 상속 구조

`tsconfig.base.json`에 strict 옵션과 `moduleResolution: "Bundler"`를 두고
각 패키지/앱이 상속한다. Bundler 해석 모드 덕분에 소스 간 `.js` 확장자
import와 `.svelte`/`.tsx` 참조가 모두 동작한다.

- packages/*: `rootDir: src`, `outDir: dist` 추가 (emit용)
- apps/*: `noEmit: true`, `types: ["vite/client"]` 추가
- react 계열: `jsx: "react-jsx"` 추가
