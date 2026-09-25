# 상태 저장/복원 (State Persistence)

그리드 상태를 직렬화해 localStorage 등에 저장하고 복원한다.

## 코어 API

```ts
const state = grid.getState();                    // GridPersistedState (직렬화 가능)
localStorage.setItem("grid-state", JSON.stringify(state));

grid.applyState(JSON.parse(localStorage.getItem("grid-state")!));

// 부분 적용도 가능 — 없는 필드는 건너뜀
grid.applyState({ sortState: saved.sortState });
```

## 저장되는 필드

| 필드 | 내용 |
| ---- | ---- |
| `sortState` | 다중 정렬 조건 |
| `filters` | 컬럼 필터 맵 |
| `searchText` | 전역 검색어 |
| `pageIndex` / `pageSize` | 페이징 |
| `groupBy` | 행 그룹화 기준 |
| `selectionMode` | 셀 선택 모드 |
| `columnState` | 컬럼 order / width / visible / pinned |

없는 컬럼·필드를 참조하는 상태는 무시되므로 스키마 변경에도 안전하다.

## React 예시 — 페이지 로드 시 복원

```tsx
const { grid } = useGridCore({ columns, data });

useEffect(() => {
  const saved = localStorage.getItem("grid-state");
  if (saved) grid.applyState(JSON.parse(saved));

  return grid.subscribe(() => {
    localStorage.setItem("grid-state", JSON.stringify(grid.getState()));
  });
}, [grid]);
```

Vue/Svelte도 동일 패턴 — 마운트 시 `applyState`, `grid.subscribe` 안에서
`getState` 저장.
