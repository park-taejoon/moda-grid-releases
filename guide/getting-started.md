# 시작하기

## 패키지 선택

| 사용 환경 | 설치할 것 |
| --------- | --------- |
| React 앱 | `@moda-grid/react` |
| Vue 3 앱 | `@moda-grid/vue` |
| Vue 2.7 앱 | `@moda-grid/vue2` (Vue 2.6 이하 미지원) |
| Svelte 5 앱 | `@moda-grid/svelte` |
| 빌드 도구 없는 페이지 / CSP 제약 환경 | CDN `moda-grid.js` + `style.css` |
| 프레임워크 없이 직접 렌더링 | `@moda-grid/core` |

어댑터 패키지는 `@moda-grid/core`를 자동으로 포함한다.

## 요구사항

- Node.js >= 20, pnpm >= 9 (모노레포 개발 환경)
- 브라우저 사용 시 별도 런타임 요구사항 없음 (ESM/IIFE)

> 개발 환경은 pnpm 전용이다 — 내부 의존성이 `workspace:*` 프로토콜과
> `pnpm-workspace.yaml`을 사용해 npm이 인식하지 못한다.
> **라이브러리 사용자**는 npm/yarn 등 어떤 클라이언트로도 설치 가능하다 —
> 각 패키지의 `publishConfig`가 publish 시 exports를 `dist/`로 치환한다.
> 배포: `pnpm publish:packages` 또는 `v*` 태그 push → `publish-npm.yml`.

## Hello Grid — 30초

```ts
import { GridCore } from "@moda-grid/core";

interface User { id: number; name: string; age: number; }

const grid = new GridCore<User>({
  columns: [
    { field: "id", header: "ID", width: 60 },
    { field: "name", header: "이름", filterable: true },
    { field: "age", header: "나이" },
  ],
  data: [
    { id: 1, name: "Hana", age: 42 },
    { id: 2, name: "Daeho", age: 36 },
  ],
});

grid.subscribe(() => console.log(grid.getSnapshot().rows));
grid.toggleSort("age"); // → 나이 오름차순으로 스냅샷 갱신
```

## 핵심 개념 3가지

### 1. 헤드리스 코어

`GridCore`는 DOM을 모른다. 상태(정렬·필터·페이징·선택·편집…)와 파생 데이터
계산만 한다. DOM 렌더링은 어댑터나 사용자 코드의 몫이다.

### 2. 스냅샷 + 구독

```ts
const unsubscribe = grid.subscribe(() => {
  const snap = grid.getSnapshot(); // 메모이즈된 불변 스냅샷
  render(snap);
});
```

- 모든 뮤테이션(`setData`, `toggleSort`, `setFilter` …)은 내부에서
  `notify()`를 호출해 스냅샷을 재계산하고 구독자에게 알린다.
- 스냅샷은 **메모이즈** — 변경 없으면 같은 참조를 반환한다
  (React `useSyncExternalStore` 요구사항).
- 스냅샷 필드 전체 목록: [02-core-api.md](../02-core-api.md) 의
  "반응형 어댑터 계약" 참고.

### 3. 데이터 파이프라인

```
rawData → 필터(AND) → 전역 검색 → 정렬 → 페이징 → visibleData
```

`snapshot.rows`(=`visibleData`)는 이 파이프라인을 통과한 최종 출력이다.
필터/검색 변경 시 `pageIndex`는 0으로 리셋된다.

## 데모 앱으로 확인하기

소스 레포(`moda-grid`)를 클론했다면:

```bash
pnpm install
pnpm dev:react    # http://localhost:5173
pnpm dev:vue      # http://localhost:5174 (Vue 3)
pnpm dev:svelte   # http://localhost:5175 (Svelte 5)
pnpm dev:vue2     # http://localhost:5176 (Vue 2.7)
pnpm test         # 코어 vitest
pnpm bench        # 성능 벤치마크
```

## 다음 단계

- 사용 중인 프레임워크의 [플랫폼 가이드](./README.md#플랫폼별-가이드)로 이동
- **IBSheet 사용자**는 [ibsheet-migration.md](./ibsheet-migration.md)의
  개념 매핑 치트시트로 바로 시작할 수 있다
- 필요한 기능의 [기능 가이드](./README.md#기능별-가이드)에서 복사 가능한 예시 확인
