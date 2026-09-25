# 빌드 & 개발

## 명령어

| 명령 | 위치 | 동작 |
| ---- | ---- | ---- |
| `pnpm install` | 루트 | 워크스페이스 전체 설치 |
| `pnpm build` | 루트 | `packages/*` 빌드 — pnpm이 의존 순서 자동 정렬 |
| `pnpm build:apps` | 루트 | dev 앱 프로덕션 빌드 (타입체크 포함) |
| `pnpm build:all` | 루트 | 패키지 + 앱 전체 |
| `pnpm typecheck` | 루트 | 전체 프로젝트 타입 검사 |
| `pnpm test` | 루트 | `packages/core` vitest 실행 |
| `pnpm dev` | 루트 | dev 앱 3개 병렬 실행 |
| `pnpm dev:react` / `:vue` / `:svelte` / `:vue2` | 루트 | 개별 dev 서버 (5173/5174/5175/5176) |
| `pnpm clean` | 루트 | 모든 `dist/`, `node_modules/` 제거 |
| `pnpm dev` | 패키지 | `tsc --watch` / `svelte-package --watch` |

## 빌드 체인

```
tsc                      svelte-package
core ──┬─→ react (dist)  svelte (dist)
       ├─→ vue   (dist)
       └─→ dev 앱들은 vite가 소스를 직접 변환 (source-first exports)
```

- `pnpm -r build`는 pnpm이 `workspace:*` 의존 그래프를 읽어
  **core를 먼저** 빌드한다. 순서를 따로 명시할 필요가 없다.
- dev 앱 실행에는 패키지 빌드가 **불필요**하다 — `exports`가 `src`를
  가리키므로 Vite가 소스를 직접 컴파일한다. 코어를 고치면 모든 앱에서
  HMR로 즉시 반영된다.
- `dist/`는 배포(publish)용 산출물이다. 실제 npm 배포 시에는
  `publishConfig.exports`를 dist로 가리키도록 추가할 것.

## 요구 런타임

- Node.js >= 20, pnpm >= 9 (`engines` 필드 참조)
- Vite 6 계열 사용 — Node 20.11 이하에서는 Vite 7이 동작하지 않으므로
  vite 메이저를 올릴 때 Node 최소 버전을 함께 확인한다.

## 새 프레임워크 어댑터 추가 (예: Solid)

1. `packages/solid/package.json` 생성 — `@moda-grid/core: workspace:*` +
   프레임워크를 `peerDependencies`로 선언
2. `src/index.ts`에서 `GridCore`의 `subscribe`/`getSnapshot`을 해당
   프레임워크 반응형 API로 연결 (Solid라면 `createSignal`)
3. `tsconfig.json`은 `../../tsconfig.base.json` 상속 + `rootDir/outDir`
4. `apps/dev-solid`에 Vite 앱 추가, `pnpm-workspace.yaml`은 자동 인식
5. 루트 `package.json`에 `dev:solid` 스크립트 추가

## 트러블슈팅

- **타입이 안 잡힐 때**: 어댑터가 `Grid<User>`를 `Grid<RowData>`로 넘기는
  곳에서 불변성 에러가 날 수 있다. 제네릭을 끝까지 유지할 것
  (Svelte는 `generics` 속성 사용).
- **Vue 2 관련**: Vue 2는 EOL — `@moda-grid/vue2`는 내장 Composition API가
  있는 2.7.x 전용이며 `vue` 버전을 `2.7.16`으로 고정한다. Vue 2.6 이하가
  필요하면 `@vue/composition-api` + `vite-plugin-vue2`(비공식) 조합이
  필요하고 템플릿 타입체크는 별도 도구가 없다.
- **svelte-package 관련**: `@sveltejs/package` 2.x는 Svelte 5를 지원하며
  `src/lib` → `dist`로 변환한다. `svelte.config.js`의 `vitePreprocess`가
  필요하다.
- **esbuild 스크립트 경고**: pnpm 10은 postinstall을 기본 차단한다.
  esbuild는 optional deps의 바이너리로 동작하므로 현재 구성에서는
  승인 없이 정상 동작한다.
