# 벤치마크

`pnpm bench`로 실행. 두 개의 벤치가 돈다.

1. `packages/core/src/grid.bench.ts` — 10만 행 데이터 파이프라인 순수 측정
2. `apps/bench/src/vs-aggrid.bench.ts` — 1만 행, AG Grid Community와의 비교

## 방법

- 공통 데이터: 시드 고정 랜덤 6컬럼(id/name/email/role/age/joinedAt)
- **moda-grid**: 헤드리스 코어만 측정 (DOM/렌더링 없음)
- **AG Grid Community 36.2.0**: `happy-dom` 위에 `createGrid`로 실제 그리드를
  생성해 측정 (렌더링 비용 일부 포함)
- 모든 case는 반복 시 실제 작업이 발생하도록 상태를 교대/리셋
  (no-op early-return으로 허수가 측정되지 않도록 처리)
- `happy-dom` 환경 특성상 AG Grid의 절대값은 실제 브라우저보다 불리할 수 있음.
  절대 수치보다 **경향**으로 해석할 것

## 결과 (10k 행, mean)

| 작업 | moda-grid | AG Grid Community |
|---|---:|---:|
| 초기 생성 (AG는 첫 렌더 포함) | 0.002 ms | 25.2 ms |
| 데이터 교체 (`setData` / `rowData`) | 0.002 ms | 0.77 ms |
| 정렬 토글 | 2.35 ms | 16.1 ms |
| 텍스트 필터 | 0.60 ms | 12.9 ms |
| 검색 (`setSearch` / quickFilter) | 0.52 ms | 9.2 ms |
| 셀 편집 + undo + redo | 1.08 ms | —¹ |
| 10행 갱신 (`pasteTsv`/`applyTransaction`) | 0.003 ms | 9.3 ms |
| 상태 저장 + 복원 (`getState`) | 0.68 ms | 2.10 ms |
| 컬럼 자동 너비 | 3.10 ms | 6.10 ms |
| 하단 고정 행 설정 | ~0 µs | 0.38 ms |

¹AG Grid는 `startEditingCell`이 실제 DOM 셀을 요구해 happy-dom에서 의미 있는
비교치를 얻기 어려워 생략.

### Enterprise 전용이라 비교 불가 (moda-grid는 무상 제공)

| 기능 | moda-grid |
|---|---:|
| Set 필터 (값 체크리스트) | 0.65 ms |
| 그룹화 + 집계 | 0.57 ms |
| TSV 복사/붙여넣기 (50셀) | 0.003 ms |

## 해석

- moda-grid 코어가 데이터 파이프라인 연산(정렬/필터/검색)에서 환경상 대부분
  빠르게 측정됨. 헤드리스라 렌더링 비용이 없다는 점이 가장 큰 차이.
- AG Grid 수치에는 happy-dom 렌더/모듈 디스패치 비용이 포함되어 있어 실제
  브라우저에서는 격차가 줄 수 있음.
- 기능 면에서는 상용 비교의 중요한 포인트: AG Grid의 Set 필터·그룹화·클립보드·
  범위 선택은 **Enterprise(유료)** 전용인 반면, moda-grid는 MIT로 동일 기능 제공.

## 재현

```bash
pnpm install
pnpm bench                    # core 100k + vs-AG Grid 10k
pnpm --filter bench run bench # 비교 벤치만
```
