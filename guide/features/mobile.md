# 모바일 / 터치 지원

moda-grid는 마우스·터치·펜을 Pointer Events 단일 경로로 처리하며,
HTML5 Drag&Drop이 동작하지 않는 iOS/Android에서도 커스텀 터치
드래그 경로로 동일한 기능을 제공한다. 별도 옵션 없이 터치
환경에서 자동으로 활성화된다.

## 터치에서 동작하는 상호작용

| 제스처                            | 동작                                                     |
| --------------------------------- | -------------------------------------------------------- |
| 탭                                | 셀 선택, 정렬 토글, 체크박스, 편집 진입(singleClickEdit) |
| 셀/헤더 롱프레스 (550ms)          | `contextMenu` / `headerContextMenu` 메뉴 열기            |
| 헤더 가로 드래그                  | 컬럼 재배치 (`reorderable`)                              |
| 행 핸들(⠿) 세로 드래그            | 행 재배치 (`ColumnDef.rowDrag`)                          |
| 피벗 칩 드래그                    | 존 이동 / 존 내 위치 변경 / fields 존 드롭 시 제거       |
| 헤더 → 그룹 패널 드래그           | 그룹 추가 (`groupPanel`)                                 |
| 리사이저/행 높이/fill 핸들 드래그 | 컬럼 너비·행 높이·범위 채우기                            |
| 셀 안 세로 드래그                 | 일반 스크롤 (가상 스크롤 유지)                           |

## 제스처 규칙

- **롱프레스** — 터치를 약 550ms 유지하면 메뉴가 열린다. 손가락이
  10px 이상 움직이거나 떼면 취소되고, 브라우저가 스크롤을 가로채면
  자연스럽게 취소된다. 메뉴 발화 후 손가락을 뗄 때 발생하는 compat
  click은 삼켜져 셀 활성화·편집 진입으로 이어지지 않는다.
- **터치 드래그** — 소스를 누른 채 8px 이상 움직이면 드래그로
  확정된다. 방향 제약이 있다: 컬럼 재배치는 가로, 행 재배치는 세로,
  피벗 칩은 임의 방향. 그 미만의 이동으로 떼면 일반 탭이다.
- **드롭 해석** — 손가락 아래의 실제 요소(`elementFromPoint`)로
  대상을 찾는다. 헤더는 `data-mg-field`, 피벗 존은
  `data-mg-pivot-zone`, 칩은 `data-mg-pivot-index` DOM 계약을 쓴다.
  드래그 중 리렌더로 소스 노드가 교체돼도 추적은 document 레벨이라
  계속된다.

## 스크롤과의 공존

`touch-action`은 제스처 의미를 나누는 경계다:

- `.mg-resizer`, `.mg-row-resizer`, `.mg-fill-handle`,
  `.mg-row-drag-handle`, `.mg-pivot-chip` — `none` (전용 핸들)
- `thead th` — `pan-y` (세로 스크롤은 살리고 가로 제스처만 가져옴)
- `.mg-table` — `manipulation` (일반 탭/스크롤, 더블탭 줌 억제)

따라서 셀 본문에서 세로로 미는 것은 항상 스크롤이다. 범위 선택
드래그는 터치에서 지원하지 않는다 — 스크롤과 구분할 수 없어서다.

## 터치 밀도

`@media (pointer: coarse)`에서 전용 핸들의 유효 터치 영역이 자동으로
커진다(리사이저 22px, 행 리사이저 14px, fill 핸들 14px, 행 핸들/피벗
칩 패딩 확대). 그리드는 카드형으로 변환하지 않고 가로 스크롤 표를
유지한다 — 대용량 데이터 그리드의 표준 레이아웃이다.

## 관련 API

드래그/재배치는 기존 API를 그대로 호출한다 — `reorderColumn`,
`beginRowDrag`/`updateRowDropPosition`/`endRowDrag`, `addPivotField`,
`setGroupBy`, `fillRange`, `openContextMenu`/`openHeaderContextMenu`.
코어의 `touch.ts` 유틸(`createLongPressTracker`, `createTouchDrag`)이
5개 렌더러의 공용 제스처 경로다.
