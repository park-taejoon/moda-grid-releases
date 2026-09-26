# 행 상태 추적 (I / U / D)

그리드가 각 행의 변경 상태를 추적해 **맨 왼쪽 상태 컬럼**에 표시하고, 변경분만 뽑아 서버에 저장할 수 있게 해줍니다.

## 개요

| 상태 | 의미 | 발생 시점 |
| ---- | ---- | --------- |
| `I` | Insert — 새로 입력된 행 | `addRows()`, `importCsv()`로 추가된 행 |
| `U` | Update — 수정된 행 | 셀 편집, `pasteTsv`로 값이 바뀐 행 |
| `D` | Delete — 삭제 예정 행 | `deleteRowsByIds()`로 마킹된 행 (화면에는 남아 있음) |
| 없음 | 변경 없음 | 초기 로드 상태, `commitChanges()` 후 |

**핵심 규칙**

- `I` 행은 아무리 수정해도 `I` 유지 — 서버에는 한 번의 INSERT면 충분하기 때문
- `I` 행을 삭제하면 서버에 존재하지 않는 행이므로 **즉시 제거** (D 마킹이 아님)
- `D` 행은 `commitChanges()` 전까지 그리드에 남아 있어 사용자가 확인 가능 — `restoreRowsByIds()`로 되돌릴 수 있음
- 행 식별은 `getRowId` 옵션 기준 — 서버 저장 연동 시 반드시 설정하는 것을 권장 (없으면 데이터 인덱스)

## 코어 API

```ts
// 상태 조회
grid.getRowState(row);            // "I" | "U" | "D" | undefined
grid.getChanges();                // { inserted, updated, deleted } — JSON 직렬화 가능
grid.hasChanges();                // 변경분 존재 여부

// 삭제 마킹 / 복원
grid.deleteRowsByIds(["3", "7"]); // 기존 행 → D 마킹, I 행 → 즉시 제거
grid.restoreRowsByIds(["3"]);     // D 마킹 해제

// 확정 / 취소
grid.commitChanges();             // D 행 실제 제거 + 모든 마킹 해제 (서버 저장 성공 후)
grid.clearChanges();              // 마킹만 해제 (D 행은 화면에 복귀)

// 유효성 검사 (저장 전)
grid.validateChanges();           // I/U 행의 required/validate 에러 목록
grid.validateRow(row);            // 행 하나의 에러 목록
grid.getCellError(row, col);      // 셀 하나의 에러 메시지 (없으면 null)
```

스냅샷에는 `rowStates: Record<string, RowState>`가 포함되어 어댑터가 행 클래스/상태 셀에 사용합니다.

### 서버 저장 패턴

```ts
async function save() {
  // 저장 전 검증 — required/validate 실패 셀이 있으면 전송 중단
  const errors = grid.validateChanges();
  if (errors.length > 0) {
    console.warn(errors.map((e) => `[${e.rowId}] ${e.field}: ${e.message}`));
    return;
  }
  const changes = grid.getChanges();
  await fetch("/api/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(changes),
  });
  grid.commitChanges(); // 저장 성공 → D 행 제거 + 마킹 초기화
}
```

`getChanges()`는 `{ inserted: TData[], updated: TData[], deleted: TData[] }` — 각 배열에 최신 행 객체가 들어 있어 그대로 `JSON.stringify`해 서버로 보낼 수 있습니다.

## 어댑터 — `rowStatus` prop

```tsx
<DataGrid rowStatus /* 맨 왼쪽에 "상태" 컬럼 */ />
```

```vue
<DataGrid row-status />
```

```svelte
<DataGrid rowStatus />
```

- 체크박스 컬럼(`rowCheckboxes`)보다 더 왼쪽에 위치합니다.
- 헤더에는 "상태"가 표시되고 각 행에 `I`/`U`/`D` 배지가 나타납니다.
- 그룹 행·고정 행·총계 행에는 빈 상태 셀이 들어가 컬럼 정렬이 유지됩니다.

## 스타일링 — 행/상태 셀에 색 입히기

상태별 클래스가 자동으로 부여되므로 CSS만 추가하면 됩니다.

| 대상 | 클래스 |
| ---- | ------ |
| 행 전체 | `mg-row-I`, `mg-row-U`, `mg-row-D` |
| 상태 셀 | `mg-status-cell` + `mg-status-I`, `mg-status-U`, `mg-status-D` |
| 검증 실패 셀 | `mg-cell-invalid` (변경 행에서 required/validate 실패 시) |

```css
/* 예: 수정된 행 전체를 노랗게, 삭제 예정 행은 취소선 */
.mg-row-U > td { background: #fef9c3; }
.mg-row-D > td { background: #fee2e2; text-decoration: line-through; }
.mg-row-I > td { background: #dcfce7; }

/* 상태 배지만 색상 */
.mg-status-I { color: #16a34a; }
.mg-status-U { color: #d97706; }
.mg-status-D { color: #dc2626; }
```

`getRowClass` 옵션으로도 `snapshot.rowStates`를 참조해 커스텀 클래스를 줄 수 있습니다.

## 주의 사항

- `setData()`로 데이터를 통째로 교체하면 행 상태도 초기화됩니다 — 서버에서 다시 로드한 시점이 곧 "깨끗한 상태"라는 의도된 동작입니다.
- `commitChanges()`는 **로컬 마킹 정리만** 합니다. 서버 호출은 애플리케이션 코드에서 처리하세요.
- 서버 사이드 모드(`serverSide`)에서는 로컬 행 추적과 충돌할 수 있으므로 사용을 권장하지 않습니다.
- Undo/Redo는 데이터 변경을 되돌리지만 행 상태 마킹까지 완벽히 복원하지는 않습니다 — 저장 직전 `getChanges()` 결과를 기준으로 삼으세요.
