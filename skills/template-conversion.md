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

在处理 `<template>` 模板时，遇到以下形式的组件标签，**识别为 Huadesign 组件库的组件**，需要优先参考 `huadesignVue转化规则.md` 进行属性转化：

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

---

## 10. 子模块公共组件处理规则

### 10.1 背景

Git submodule 目录（如 `src/components/`）中的公共组件**不参与本项目的迁移**。但业务文件的 `<template>` 中仍会引用这些来自子模块的自定义公共组件，模板转化时需要正确处理它们。

### 10.2 组件识别规则

本 Skill 从调用参数中接收 `submodule_components` 映射表（由迁移专家在 4.0 前置阶段建立），格式如下：

```json
{
  "submodule_components": {
    "DeleteConfirm": { "path": "src/components/deleteConfirm.vue", "tag": "delete-confirm" },
    "PageHeader": { "path": "src/components/pageHeader.vue", "tag": "page-header" },
    "SearchForm": { "path": "src/components/searchForm.vue", "tag": "search-form" }
  }
}
```

**识别算法**：
1. 遍历 `<template>` 中的所有非原生 HTML 标签
2. 将每个标签名（kebab-case 格式）与 `submodule_components` 中的每个 `tag` 字段匹配
3. 匹配成功 → 识别为**子模块公共组件**，按 10.3 节规则处理
4. 匹配失败 → 按普通组件（Huadesign 或其他）继续处理

### 10.3 转化规则

对于识别为子模块公共组件的标签，执行以下规则：

#### 10.3.1 默认规则：保持属性原样（禁止自动转化）

**核心原则：默认不对子模块公共组件的任何属性和事件做转化。**

- `.sync` 修饰符：**禁止**转化为 `v-model:prop`，保持 `:prop.sync="value"` 的写法不变
- `v-model`：**禁止**转化为 Vue3 的 `modelValue`/`update:modelValue`，保持原样
- `.native` 修饰符：**保留**，不允许移除
- 所有属性和事件：保持 Vue2 原始写法，不做任何语法升级

**原因**：
- 子模块公共组件本身不参与迁移，它们在 Vue3 环境中仍保持 Vue2 的接口规范
- 如果将 `.sync` 转化为 `v-model:prop`，会导致子模块组件接收不到正确的属性/事件而功能异常
- 保持原样确保与子模块的兼容性

#### 10.3.2 例外规则：差异文档覆盖

当本 Skill 同时接收到来自迁移专家的 `公共组件库差异文档.md` 中对应组件的规则片段时，按以下优先级判断：

1. **差异文档有明确规则** → 按差异文档的规则转化
2. **差异文档未覆盖的属性** → 保持原样（按 10.3.1 默认规则处理）

差异文档传入格式示例：

```json
{
  "diff_rules": {
    "delete-confirm": {
      "transformations": [
        { "from": ":visible.sync", "to": "v-model:visible" }
      ]
    }
  }
}
```

只有当差异文档明确说明某组件的某属性需要从 A→B 转化时，才对该属性执行转化。

#### 10.3.3 判断逻辑流程图

```
[识别到子模块公共组件标签]
  → 检查是否有该组件的 diff_rules
  ├── 有差异规则 → 按规则文档转化对应属性，未覆盖属性保持原样
  └── 无差异规则 → 所有属性保持原样（不做任何转化）
```

### 10.4 与 Huadesign 组件的区别

| 维度 | Huadesign 组件 | 子模块公共组件 |
|------|---------------|--------------|
| 识别方式 | 标签名前缀 `a-` | 匹配 `submodule_components` 映射表 |
| 转化策略 | 按规则文档执行转化，未覆盖回退通用规则 | 默认保持原样，仅差异文档明确规则时才转化 |
| 规则来源 | `huadesignVue转化规则.md` | `公共组件库差异文档.md` |
| 处理优先级 | Huadesign 优先于通用规则 | 差异文档优先，通用规则不适用 |

### 10.5 日志记录

对每个识别到的子模块公共组件，在迁移日志中记录：

```json
{
  "type": "SUBMODULE_COMPONENT",
  "component": "delete-confirm",
  "source": "src/components/deleteConfirm.vue",
  "action": "KEEP_AS_IS",
  "reason": "子模块公共组件，不参与迁移，保持属性原样",
  "diff_rules_applied": false
}
```

当有差异规则被应用时：

```json
{
  "type": "SUBMODULE_COMPONENT",
  "component": "delete-confirm",
  "source": "src/components/deleteConfirm.vue",
  "action": "PARTIAL_TRANSFORM",
  "reason": "公共组件库差异文档有覆盖规则",
  "diff_rules_applied": true,
  "transformations": [
    { "attribute": ":visible.sync", "from": ":visible.sync=\"val\"", "to": "v-model:visible=\"val\"" }
  ]
}
```
