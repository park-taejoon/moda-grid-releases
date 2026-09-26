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
- `dist/`는 배포(publish)용 산출물이다. 각 패키지의 `publishConfig.exports`가
  dist를 가리키므로 **publish 시에는 자동으로 dist 경로가 적용**되고
  `workspace:*` 의존성도 실제 버전으로 변환된다.

## npm 배포

### 릴리즈 (권장) — `pnpm release`

`scripts/release.mjs`가 **5개 패키지의 version과 git 태그를 한 번에** 맞춘다 —
태그와 package.json 버전 불일치로 배포가 실패하는 일이 없다:

```bash
pnpm release patch        # 0.1.0 → 0.1.1 : 버전 일괄 갱신 + 커밋 + 태그 v0.1.1
pnpm release minor        # → 0.2.0
pnpm release major        # → 1.0.0
pnpm release 0.3.0        # 명시적 버전
pnpm release patch --push # 커밋 + 태그 + origin push까지 한 번에
```

동작 순서:

1. `packages/*/package.json`의 `version`을 전부 새 버전으로 갱신
   (버전 불일치가 있으면 경고 후 통일)
2. 갱신분만 `chore(release): vX.Y.Z`로 커밋 — 작업 중인 다른 변경분은 건드리지 않음
3. `git tag vX.Y.Z` 생성 (이미 있으면 중단)
4. `--push`가 있으면 커밋+태그를 origin에 push → `publish-npm.yml` 실행

push 없이 태그만 만들었다면 이후에 `git push && git push origin vX.Y.Z`로
배포를 마무리한다.

### 수동 배포

```bash
pnpm build               # dist 생성 (css/SFC 복사 포함)
pnpm publish:packages    # packages/* 일괄 publish (이미 올라간 버전은 스킵)
```

`v*` 태그 push 시 `publish-npm.yml` 워크플로우가 typecheck → test →
build → publish를 실행한다. 필요한 Secret: `NPM_TOKEN`(npm Automation 토큰,
`@moda-grid` 스코프에 publish 권한 필요).
npm 페이지에 표시되는 README는 각 `packages/*/README.md`.
배포되는 tarball은 `files: ["dist"]`로 제한되어 dist/README/LICENSE만 포함된다.

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
