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
}
```

## 너비 리사이즈

`resizable` prop(기본값 `true`)으로 헤더 우측 경계에 `.mg-resizer` 핸들이
표시된다. 드래그는 `requestAnimationFrame`으로 스로틀해 프레임당 최대 1회
코어에 반영하고, 포커스된 핸들에서 `←`/`→` 키로 ±10px 조절 가능하다.

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

## 런타임 컬럼 교체

```ts
grid.setColumns(newColumns);   // 너비/순서 상태 유지, 새 컬럼은 끝에 추가
```

`snapshot.columnState`(order 오름차순 `ColumnState[]`)로 레이아웃 상태를
직접 읽을 수 있다. 레이아웃 변경은 파이프라인을 재계산하지 않고 스냅샷만
갱신해 리사이즈/드래그 비용을 줄인다.
