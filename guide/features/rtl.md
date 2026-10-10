# RTL (오른쪽→왼쪽)

`dir="rtl"` 옵션/prop으로 그리드를 RTL로 렌더한다. 루트 요소에
`dir="rtl"`이 붙고 방향 의존 동작이 시각 방향을 따라간다.

```tsx
<DataGrid dir="rtl" columns={columns} data={rows} />
```

```ts
const grid = new GridCore({ columns, data, dir: "rtl" });
grid.setDir("ltr"); // 런타임 전환 — snapshot.dir이 갱신된다
```

## 동작 계약

| 영역             | RTL 동작                                                                          |
| ---------------- | --------------------------------------------------------------------------------- |
| 루트             | `dir="rtl"` 속성 — CSS 논리 속성·텍스트 방향이 자동 반전                          |
| 화살표 키        | `ArrowLeft`/`ArrowRight`는 **시각** 방향 — ArrowLeft가 다음 컬럼으로 이동         |
| Ctrl+Arrow       | 시각적 가장자리 — Ctrl+←는 화면 왼쪽 끝(논리 마지막 컬럼)                         |
| Home/End·Tab     | 논리 방향 유지 — Tab은 항상 다음 컬럼(인덱스 증가)으로                            |
| pinned 컬럼      | `pinned: "left"`는 inline-start — RTL에서는 **우측**에 고정                       |
| 컬럼 가상화      | `scrollLeft`는 논리 좌표로 정규화 — 음수 규약(최신)·양수-역방향(레거시) 모두 수용 |
| 트리/그룹 인덴트 | `padding-inline-start` — 시작쪽(우측)으로 들여쓰기                                |

## scrollLeft 정규화

브라우저마다 RTL `scrollLeft` 규약이 다르다:

- **최신 규약**(Chrome/Firefox/현행 Safari): 시작에서 `0`, 끝으로 갈수록 **음수**
- **레거시**(구형 Safari): 끝에서 `0`, 시작에서 `max`(양수)

코어의 컬럼 가상화는 항상 논리 좌표(`0` = inline-start, 양수로 증가)로
동작하고, `domToLogicalScrollLeft`/`logicalToDomScrollLeft` 헬퍼가 어댑터
경계에서 변환한다. 어댑터 `onScroll`은 DOM 값을 그대로 `handleScroll`에
넘기고, `snapshot.virtualCols.scrollLeft`를 DOM에 쓸 때만 역변환한다.

```ts
import {
  domToLogicalScrollLeft,
  logicalToDomScrollLeft,
} from "@moda-grid/core";

// DOM → 논리 (스크롤 이벤트): maxScroll = scrollWidth - clientWidth
const logical = domToLogicalScrollLeft(el.scrollLeft, true, maxScroll);
// 논리 → DOM (프로그램 스크롤 반영)
el.scrollLeft = logicalToDomScrollLeft(vc.scrollLeft, true);
```

`scrollToColumn`/`ensureColumnVisible`은 논리 좌표를 생성하므로 RTL에서도
별도 처리 없이 올바른 위치로 스크롤한다.
