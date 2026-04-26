# 模板语法转化 Skill - Template Conversion

## 功能描述
将 Vue2 模板语法转化为 Vue3 兼容的模板语法。处理插槽语法升级、指令转化、过渡类名更新、**Huadesign 组件库属性转化**等模板层面的迁移工作。

## 触发条件
当迁移专家处理 `.vue` 文件中的 `<template>` 部分时调用。

---

## 1. 插槽语法转化

### 1.1 具名插槽

```vue
<!-- Vue2 -->
<template slot="header">
  <div>Header</div>
</template>

<!-- Vue3 -->
<template v-slot:header>
  <div>Header</div>
</template>
<!-- 或缩写 -->
<template #header>
  <div>Header</div>
</template>
```

### 1.2 作用域插槽

```vue
<!-- Vue2 -->
<template slot-scope="{ row }">
  <span>{{ row.name }}</span>
</template>

<!-- Vue3 -->
<template v-slot:default="{ row }">
  <span>{{ row.name }}</span>
</template>
<!-- 或缩写 -->
<template #default="{ row }">
  <span>{{ row.name }}</span>
</template>
```

### 1.3 同时使用具名和作用域

```vue
<!-- Vue2 -->
<template slot="item" slot-scope="{ data }">
  <span>{{ data }}</span>
</template>

<!-- Vue3 -->
<template #item="{ data }">
  <span>{{ data }}</span>
</template>
```

---

## 2. v-model 模板语法

### 2.1 组件 v-model 变化

```vue
<!-- Vue2（默认行为） -->
<MyComponent v-model="value" />
<!-- 等价于 :value="value" @input="value = $event" -->

<!-- Vue3（默认行为） -->
<MyComponent v-model="value" />
<!-- 等价于 :modelValue="value" @update:modelValue="value = $event" -->
```

### 2.2 .sync → v-model 参数

```vue
<!-- Vue2 -->
<el-dialog :visible.sync="dialogVisible" />

<!-- Vue3 -->
<el-dialog v-model:visible="dialogVisible" />
```

### 2.3 多个 v-model

```vue
<!-- Vue2：使用 .sync 实现多值绑定 -->
<Component :name.sync="name" :visible.sync="visible" />

<!-- Vue3：多个 v-model -->
<Component v-model:name="name" v-model:visible="visible" />
```

---

## 3. 事件修饰符处理

### 3.1 .native 修饰符移除

```vue
<!-- Vue2 -->
<ChildComponent @click.native="handleClick" />

<!-- Vue3（移除 .native，默认所有事件为原生事件） -->
<ChildComponent @click="handleClick" />
```

### 3.2 其他事件修饰符（保持不变）

以下修饰符在 Vue3 中行为不变：
- `.stop` → `@click.stop="fn"`
- `.prevent` → `@submit.prevent="fn"`
- `.capture` → `@click.capture="fn"`
- `.self` → `@click.self="fn"`
- `.once` → `@click.once="fn"`
- `.passive` → `@scroll.passive="fn"`
- 按键修饰符：`.enter`, `.tab`, `.delete`, `.esc`, `.space`, `.up`, `.down`, `.left`, `.right`

---

## 4. 过渡类名变化

| Vue2 | Vue3 |
|------|------|
| `v-enter` | `v-enter-from` |
| `v-leave` | `v-leave-from` |
| `v-enter-active` | `v-enter-active`（不变） |
| `v-leave-active` | `v-leave-active`（不变） |
| `v-enter-to` | `v-enter-to`（不变） |
| `v-leave-to` | `v-leave-to`（不变） |

```vue
<!-- Vue2 过渡类名 -->
<style>
.fade-enter, .fade-leave { opacity: 0 }
</style>

<!-- Vue3 过渡类名 -->
<style>
.fade-enter-from, .fade-leave-from { opacity: 0 }
</style>
```

---

## 5. key 属性处理

### 5.1 v-if/v-else 建议加 key

```vue
<!-- Vue2（不强制） -->
<div v-if="type === 'A'">A</div>
<div v-else>B</div>

<!-- Vue3（建议加 key 以确保正确复用） -->
<div v-if="type === 'A'" key="type-a">A</div>
<div v-else key="type-b">B</div>
```

### 5.2 v-for 中的 key

```vue
<!-- Vue2（可以用 index） -->
<li v-for="(item, index) in list" :key="index">{{ item }}</li>

<!-- Vue3（推荐使用唯一 id） -->
<li v-for="item in list" :key="item.id">{{ item.name }}</li>
```

---

## 6. v-for + ref 变化

```vue
<!-- Vue2 -->
<div v-for="item in list" ref="items">...</div>
<!-- this.$refs.items → DOM 数组 -->

<!-- Vue3 -->
<div v-for="item in list" :ref="(el) => setItemRef(el, item.id)">...</div>
<!-- 需要使用函数形式 ref -->
<script setup>
const itemRefs = ref([])
const setItemRef = (el, id) => {
  if (el) itemRefs.value.push(el)
}
</script>
```

---

## 7. 组件事件命名规范

Vue3 中推荐使用 **kebab-case** 命名组件事件：

```vue
<!-- 推荐 -->
<template>
  <button @click="$emit('update-value')">更新</button>
</template>

<!-- 避免驼峰 -->
<template>
  <!-- Vue3 中驼峰事件可能匹配失败 -->
  <button @click="$emit('updateValue')">更新</button>
</template>
```

---

## 8. template 根节点变化

Vue3 支持 **Fragment**（多根节点），可移除不必要的包裹 div：

```vue
<!-- Vue2（必须单根节点） -->
<template>
  <div>
    <header>...</header>
    <main>...</main>
    <footer>...</footer>
  </div>
</template>

<!-- Vue3（支持多根节点） -->
<template>
  <header>...</header>
  <main>...</main>
  <footer>...</footer>
</template>
```

 **注意**：对于使用了 `v-for`、`v-if`/`v-else` 或 CSS 选择器依赖父容器的情况，仍需保留包裹元素。

---

## 9. Huadesign 组件识别与属性转化

### 9.1 组件识别规则

在处理 `<template>` 模板时，遇到以下形式的组件标签，**识别为 Huadesign 组件库的组件**，需要参考 `huadesignVue转化规则.md` 进行属性转化：

| 组件前缀 | 示例 | 说明 |
|---------|------|------|
| `a-` | `<a-modal>`, `<a-tooltip>`, `<a-button>`, `<a-table>`, `<a-form>`, `<a-input>`, `<a-select>`, `<a-drawer>`, `<a-popconfirm>`, `<a-dropdown>`, `<a-menu>`, `<a-tree>`, `<a-tabs>` 等 | Huadesign 组件库前缀 |

**识别算法**：
1. 遍历 `<template>` 中的所有非原生 HTML 标签
2. 匹配标签名是否以 `a-` 开头（区分大小写，应为小写 `a-`）
3. 符合条件则标记为 Huadesign 组件，立即暂停通用模板转化，转到 `huadesignVue转化规则.md` 查找该组件的迁移规则

### 9.2 转化流程

当识别到 Huadesign 组件后，执行以下流程：

```
[识别 a- 前缀组件]
  → 暂停通用模板转化（如 .sync → v-model）
  → 查阅 huadesignVue转化规则.md 中对应组件的规则
  → 按规则文档逐条转化该组件的属性和事件
  → 其他非 Huadesign 的通用语法仍按本 Skill 通用规则处理
  → 记录组件转化明细到迁移日志
```

**关键原则**：
1. **规则文档优先级高于通用规则**：对于 `huadesignVue转化规则.md` 中明确列出的属性或事件，严格按照规则文档执行
2. **未覆盖的属性回退通用规则**：对于规则文档中未明确列出的属性，降级使用本 Skill 的通用规则（如 `.sync` → `v-model:prop`）
3. **整组件处理**：遇到 `a-` 组件时，将该组件标签上的**所有属性和事件**作为一个整体进行处理，优先按规则文档转化，规则文档未覆盖的部分再用通用规则补充

### 9.3 属性分析示例

假设 `huadesignVue转化规则.md` 中对 `a-modal` 组件定义如下规则（示例）：

| 属性/事件 | Vue2 用法 | Vue3 用法 |
|-----------|----------|-----------|
| `visible` | `:visible.sync="val"` | `v-model:visible="val"` |
| `title` | `:title="str"` | `title="str"` 或 `:title="str"`（不变） |
| `@ok` | `@ok="fn"` | `@ok="fn"`（不变） |
| `@cancel` | `@cancel="fn"` | `@cancel="fn"`（不变） |

实际转化时，**必须**根据 `huadesignVue转化规则.md` 的实际内容执行转化，以上仅作格式示意。

### 9.4 日志记录

对每个经过 Huadesign 规则处理的 a- 组件，在迁移日志中记录：

```json
{
  "type": "HUADESIGN_COMPONENT",
  "component": "a-modal",
  "rules_source": "huadesignVue转化规则.md",
  "transformations": [
    {
      "attribute": ":visible.sync",
      "from": ":visible.sync=\"dialogVisible\"",
      "to": "v-model:visible=\"dialogVisible\"",
      "rule_ref": "huadesignVue转化规则.md §a-modal.visible"
    }
  ]
}