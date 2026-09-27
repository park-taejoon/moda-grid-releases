# 컬럼 조작 (Columns)

리사이즈, 순서 변경, 고정, 표시/숨김, 자동 너비, 그룹 헤더, 컬럼 관리 UI.

## 컬럼 정의 (`ColumnDef`) 레이아웃 필드

```ts
{
  field: "name",
  header: "이름",
  width: 140,                    // 고정 너비 px
  minWidth: 60,                  // 리사이즈 최소 (기본값 40)
  maxWidth: 300,                 // 리사이즈 최대 (기본값 무제한)
  resizable: true,               // 기본값 true
  visible: true,                 // 기본값 true
  pinned: "left",                // "left" | "right" | null
  group: "score",                // 컬럼 그룹 ID — 다단계 헤더
  align: "right",                // 셀 텍스트 정렬 left|center|right
  headerAlign: "center",         // 헤더 정렬 — 생략 시 align을 따라감
  headerTooltip: "사원 성명",     // 헤더 마우스오버 툴팁 (th title)
}
```

`align`은 셀에 `mg-align-*` 클래스, `headerAlign`은 헤더에
`mg-halign-*` 클래스로 적용된다 — 숫자 컬럼 우측 정렬, 체크박스 가운데
정렬 등에 사용한다. `headerTooltip`은 헤더 셀의 `title` 속성으로
렌더링되어 마우스 오버 시 브라우저 기본 툴팁이 표시된다.

## 너비 리사이즈

`resizable` prop(기본값 `true`)으로 헤더 우측 경계에 `.mg-resizer` 핸들이
표시된다. 드래그는 `requestAnimationFrame`으로 스로틀해 프레임당 최대 1회
코어에 반영하고, 포커스된 핸들에서 `←`/`→` 키로 ±10px 조절 가능하다.
**핸들 더블클릭은 `autoSizeColumn`** — 내용 기준 자동 너비로 즉시 조정된다
(엑셀/타사 그리드 경계 더블클릭과 동일).

테이블은 **`table-layout: fixed` + `<colgroup>`**로 너비를 관리한다 —
nowrap 셀 내용이 컬럼을 밀어내지 않으므로 좁히는 리사이즈도 정확히 동작하고
넘치는 텍스트는 말줄임(`…`)으로 잘린다. 테이블 `min-width`는 전체 컬럼
너비 합(미지정 컬럼은 120px 기본값)이라 컨테이너보다 좁아지지 않고 가로
스크롤이 생긴다 — 리사이즈 결과가 임의로 뭉개지지 않는다.

```ts
grid.setColumnWidth("name", 180);            // min/max 클램프 적용
grid.autoSizeColumn("name");                 // 내용 기준 자동 너비 (단일)
grid.autoSizeAllColumns();                   // 전체 표시 컬럼
grid.autoSizeAllColumns((t) => ctx.measureText(t).width);  // canvas 정확 측정
grid.resetColumnLayout();                    // 너비/순서를 정의 기본값으로
```

`measureText` 미지정 시 문자 길이 기반 추정(`length * 8 + 패딩`)이라
DOM 없이도 동작한다.

## 순서 변경

`reorderable` prop(기본값 `true`)으로 헤더 셀에 HTML5 DnD가 연결된다 —
드롭 대상은 `.mg-col-dragover`로 표시.

```ts
grid.reorderColumn("name", "role");   // draggedId → targetId 위치로
```

## 고정 (Pinning)

```ts
{ field: "id", pinned: "left" }        // 또는 grid.setColumnPinned("id", "right")
```

- `snapshot.visibleColumns`가 `[left 고정 → 비고정 → right 고정]` 순서로
  정렬된다 — 어댑터는 그대로 렌더링.
- `snapshot.pinOffsets`(필드→px)에 sticky 오프셋이 계산되어 들어간다.
  리사이즈 결과도 반영되고 숨긴 컬럼은 제외.
- 고정 셀은 `position: sticky` + 오프셋, 경계에 `.mg-pin-left-edge`/
  `.mg-pin-right-edge` 구분선(box-shadow), `.mg-pinned`는 불투명 배경.
- 정확한 오프셋을 위해 고정 컬럼은 `width` 지정 권장 (미지정 시 120px).

## 표시/숨김 + 컬럼 관리 UI

```ts
grid.setColumnVisible("age", false);   // = setColumnVisibility
grid.setAllColumnsVisible(true);       // 전체 토글 (전부 숨김도 허용)
```

`columnController` prop을 주면 그리드 우상단에 `컬럼 ▾` 버튼이 오버레이되고,
체크박스 팝오버(`.mg-colctl-panel`)로 전체 선택/해제와 개별 토글을 제공한다.
패널은 오버레이 클릭이나 Escape로 닫힌다.

## 그룹 헤더 (다단계 헤더)

```ts
new GridCore({
  columns: [
    { field: "name", header: "이름" },                  // 그룹 없음 → rowspan=2
    { field: "kor",  header: "국어", group: "score" },
    { field: "eng",  header: "영어", group: "score" },
    { field: "math", header: "수학", group: "score" },
  ],
  columnGroups: [{ id: "score", header: "성적" }],     // 생략 시 ID가 라벨
  data,
});
```

- `snapshot.headerGroups: HeaderGroupSpan[] | null`에 상단 행의 병합 정보
  (`{ groupId, header, startIndex, colSpan }`)가 계산된다.
- 같은 그룹의 **인접한 표시 컬럼만** 병합. 숨긴 컬럼은 제외되고 그로 인해
  인접해진 같은 그룹은 합쳐진다.
- 어댑터는 그룹 셀 `colspan`+`.mg-colgroup`, 단일 컬럼 `rowspan=2`로
  2단 `<tr>`을 렌더링한다.

## 헤더 높이 (`headerHeight`)

헤더 영역 전체 높이를 px로 지정한다 — 멀티레벨 헤더(다단계)에서는
**헤더 행 수로 균등 분할**된다 (예: `headerHeight: 60` + 2행 헤더 →
각 행 30px).

```tsx
<DataGrid headerHeight={48} />          // prop (React/Vue/Svelte 공통)
mountGrid(el, { columns, data, headerHeight: 48 });
grid.setHeaderHeight(80);               // 런타임 변경 (null이면 기본값)
```

스냅샷에 `snapshot.headerHeight: number | null`로 노출되며, 각 헤더
`tr`의 인라인 `height`로 적용된다. 지정하지 않으면 CSS 기본 높이를
따른다.

## 헤더 커스텀 클래스 (`headerClass`)

헤더 `th`에 커스텀 클래스를 추가한다 — 문자열 또는 컬럼을 받는 함수:

```ts
{ field: "name", headerClass: "th-primary" }
{ field: "age", headerClass: (col) => col.sortable === false ? "th-nosort" : null }
```

`grid.getHeaderClass(col)`로 해석된 값을 4개 어댑터와 `mountGrid`가
`th` 클래스에 병합한다.

## 런타임 컬럼 교체

```ts
grid.setColumns(newColumns);   // 너비/순서 상태 유지, 새 컬럼은 끝에 추가

// 부분 수정 — 나머지 속성은 유지한 채 필요한 것만 갱신
grid.updateColumn("name", { header: "성명", editable: false });
```

`updateColumn(field, patch)`은 대상 컬럼이 없으면 `false`를 반환하고,
`field` 자체는 patch로 덮을 수 없다. 피벗 모드에서는 원본(base) 컬럼이
대상이 된다.

### 컬럼 추가/제거

```ts
grid.addColumn({ field: "memo", header: "비고" });      // 끝에 추가
grid.addColumn({ field: "memo", header: "비고" }, 0);   // 첫 위치에 삽입
grid.removeColumn("name");                             // false = 없는 컬럼
```

`addColumn(def, index)`의 index는 **표시 순서**에도 반영된다 — 기존 컬럼의
order를 밀어내고 해당 위치에 배치한다. `removeColumn(field)`는 정렬·필터·
컬럼 상태에서 해당 필드를 함께 정리한다. 피벗 모드에서는 baseColumns가
대상이다.

`snapshot.columnState`(order 오름차순 `ColumnState[]`)로 레이아웃 상태를
직접 읽을 수 있다. 레이아웃 변경은 파이프라인을 재계산하지 않고 스냅샷만
갱신해 리사이즈/드래그 비용을 줄인다.
