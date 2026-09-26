# 프로젝트 구조

```
moda-grid/
├── package.json            # 루트 스크립트, 공유 typescript
├── pnpm-workspace.yaml     # packages/*, apps/* 워크스페이스 선언
├── tsconfig.base.json      # 모든 패키지가 상속하는 TS 공통 설정
├── packages/
│   ├── core/               # @moda-grid/core   — 순수 TS 핵심 로직
│   │   └── src/
│   │       ├── types.ts    #   ColumnDef, GridSnapshot, PageState, GridOptions …
│   │       ├── grid.ts     #   GridCore 클래스 구현 (rawData/visibleData/subscribe/notify)
│   │       ├── virtualScroll.ts # computeVirtualScroll() 순수 계산기
│   │       ├── grid.test.ts#   vitest 테스트 (GridCore API)
│   │       ├── virtualScroll.test.ts # 가상 스크롤 계산 + 통합 테스트
│   │       ├── styles.css  #   공통 .mg-* 스타일 (단일 소스)
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
│   └── dev-vue2/           # Vite + Vue 2.7 데모 (포트 5176)
└── docs/
```

## 패키지 의존 관계

```
@moda-grid/core  (의존성 없음)
@moda-grid/react    → @moda-grid/core (workspace:*), peer: react, react-dom
@moda-grid/vue      → @moda-grid/core (workspace:*), peer: vue >= 3.3
@moda-grid/svelte   → @moda-grid/core (workspace:*), peer: svelte ^5
@moda-grid/vue2     → @moda-grid/core (workspace:*), peer: vue ^2.7

dev-react   → @moda-grid/react (workspace:*)
dev-vue     → @moda-grid/vue   (workspace:*)
dev-svelte  → @moda-grid/svelte(workspace:*)
dev-vue2    → @moda-grid/vue2  (workspace:*), vue 2.7.16 고정
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
