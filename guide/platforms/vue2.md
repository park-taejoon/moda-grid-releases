# Vue 2.7 가이드 — `@moda-grid/vue2`

> **주의**: Vue 2는 2023-12-31 EOL. 이 패키지는 Composition API와
> `<script setup>`을 내장한 **Vue 2.7.x만** 지원한다 — 2.6 이하는
> `@vue/composition-api` 플러그인이 별도로 필요해 지원하지 않는다.

## 설치

```bash
npm install @moda-grid/vue2 vue@2.7.16   # vue는 2.7.x로 고정
```

Vite 앱에서는 `@vitejs/plugin-vue2`(비공식 계열 포함)가 필요하다.

## 기본 사용

```vue
<script setup lang="ts">
import { DataGrid, useGrid } from "@moda-grid/vue2";
import "@moda-grid/vue2/styles.css";      // 필수 — 공통 스타일

const { grid, state } = useGrid({ columns, data: users });
</script>

<template>
  <input placeholder="검색…" @input="grid.setSearch($event.target.value)" />
  <DataGrid
    :grid="grid"
    :columns="columns"
    :height="480"
    :row-height="37"
    column-controller
  >
    <template #cell-role="{ value }">
      <span :class="`badge role-${value}`">{{ value }}</span>
    </template>
  </DataGrid>
</template>
```

앱 진입점은 Vue 2 스타일:

```ts
import Vue from "vue";
import App from "./App.vue";

new Vue({ render: (h) => h(App) }).$mount("#app");
```

## Vue 3 어댑터와의 차이

API(`useGrid`/`useGridState`/`DataGrid` + `cell-{field}`/`editor-{field}`
슬롯 + 가상 스크롤)는 Vue 3과 **동일**하다. 다른 점:

- props 선언이 타입 기반이 아닌 **런타임 선언**(`PropType`) — Vue 2.7
  컴파일러 제약.
- 템플릿에서 `??`/`?.` 미지원 — `||`/`&&`로 대체한다.
- `vue-tsc`가 Vue 2를 지원하지 않아 타입체크는 `tsc`로 `.ts` 파일만 검사.
  `.vue` 템플릿 안의 TS `as` 캐스트는 피하고 핸들러는 script에 분리할 것.

나머지(props 표, 슬롯, 보조 UI 패턴)는 [vue3.md](./vue3.md)를 그대로 따른다.
