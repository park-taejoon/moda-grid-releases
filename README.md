# moda-grid 문서

멀티 프레임워크 데이터 그리드 라이브러리 모노레포.

## 사용자 가이드

[guide/](./guide/README.md) — 플랫폼별·기능별 사용자 가이드.
`docs/` 변경이 main에 push되면 GitHub Action(`deploy-docs.yml`)이
공개 레포 `moda-grid-releases`로 자동 배포한다.
새 기능 개발 시 문서 추가 규칙: [guide/extending.md](./guide/extending.md).

## 개발자 문서

| 문서 | 내용 |
| ---- | ---- |
| [01-project-structure.md](./01-project-structure.md) | 디렉토리 구조와 패키지 의존 관계 |
| [02-core-api.md](./02-core-api.md) | `@moda-grid/core` 헤드리스 API 레퍼런스 |
| [03-framework-adapters.md](./03-framework-adapters.md) | React / Vue 3 / Svelte 어댑터 사용법 |
| [04-build-and-development.md](./04-build-and-development.md) | 빌드·개발 명령어와 새 어댑터 추가 방법 |
| [05-usage-guide.md](./05-usage-guide.md) | 프레임워크별 임포트 사용법 + 전체 기능 명세 (CDN 포함) |
| [06-benchmark.md](./06-benchmark.md) | 성능 벤치마크 — core 측정 + AG Grid 비교 |

## 빠른 시작

```bash
pnpm install          # 워크스페이스 전체 설치
pnpm build            # packages/* 전체 빌드 (토폴로지 순서 자동)
pnpm typecheck        # 전체 타입 검사
pnpm test             # core vitest (GridCore API 테스트)

pnpm dev:react        # http://localhost:5173
pnpm dev:vue          # http://localhost:5174 (Vue 3)
pnpm dev:svelte       # http://localhost:5175
pnpm dev:vue2         # http://localhost:5176 (Vue 2.7)
pnpm dev              # 4개 앱 동시 실행
```

## 아키텍처 한눈에 보기

```
@moda-grid/core              순수 TypeScript — GridCore 클래스 (정렬·컬럼 필터·검색·페이징·선택·가상 스크롤)
   ▲        ▲        ▲        ▲
   │        │        │        │
react      vue     svelte   vue2        프레임워크별 반응형 어댑터 (얇은 래퍼)
   ▲        ▲        ▲        ▲
dev-react dev-vue dev-svelte dev-vue2   Vite 개발/데모 앱
```

핵심 로직은 전부 core의 `GridCore` 클래스에 있고, 어댑터는 `subscribe`/`getSnapshot` 계약을
각 프레임워크의 반응형 시스템(`useSyncExternalStore`, `shallowRef`, `readable`)에
연결하기만 한다. 프레임워크별 기능 동작이 항상 동일하게 유지되는 구조.
