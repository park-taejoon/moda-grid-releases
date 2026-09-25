# 고정 행 + 전체 총계 (Pinned Rows & Grand Totals)

필터·정렬과 무관하게 항상 표시되는 상/하단 행(합계·주석)과, `aggregationFn`
컬럼의 전체 총계 `<tfoot>` 행.

## 고정 행

```ts
// 초기 옵션
new GridCore({
  columns, data,
  pinnedTopRows: [{ name: "주의", role: "-", age: null }],
  pinnedBottomRows: [{ name: "합계", age: 1234 }],
});

// 런타임
grid.setPinnedTopRows(rows);      // null로 해제
grid.setPinnedBottomRows(rows);
```

스냅샷 `pinnedTopRows`/`pinnedBottomRows`(`TData[]`)에 노출되고, 어댑터는
tbody 맨 위/맨 아래에 `.mg-pinned-top-row`/`.mg-pinned-bottom-row`로
렌더링한다 (셀은 `getCellText` 기준, 선택/편집 대상 아님).

## 전체 총계 (tfoot)

`aggregationFn`이 선언된 컬럼이 하나라도 있으면 `snapshot.grandTotals`
(`Record<field, aggregate>`)에 집계가 계산되어 `<tfoot>`의 `.mg-total-row`로
표시된다:

```ts
{ field: "salary", header: "연봉", aggregationFn: "sum" }
{ field: "name",   header: "이름", aggregationFn: "count" }
```

- 집계 대상: 필터/검색을 통과한 **전체 행** (페이징 무관, 트리 자식 포함).
- 서버 사이드 모드에서는 `null` — 서버가 총계를 내려줘야 한다.
- 그룹 집계와 동일한 함수로 계산하고 `formatAggregate()`로 포맷한다
  (소수 2자리 반올림). [grouping.md](./grouping.md) 참고.
