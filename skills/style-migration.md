# 样式迁移 Skill - Style Migration

## 功能描述
将 Vue2 项目中的样式代码迁移到 Vue3 兼容格式。处理 scoped 样式适配、深度选择器转化、CSS 变量兼容、Less/Sass 样式文件迁移等工作。

## 触发条件
当迁移专家处理 `.vue` 文件中的 `<style>` 块或独立样式文件（`.less`、`.css`、`.scss`）时调用。

---

## 1. 深度选择器迁移

### 1.1 选择器语法对照

| Vue2 | Vue3 | 说明 |
|------|------|------|
| `/deep/ .child` | `:deep(.child)` | 深度选择器 |
| `>>> .child` | `:deep(.child)` | 深度选择器 |
| `::v-deep .child` | `:deep(.child)` | 深度选择器标准写法 |

### 1.2 迁移示例

```vue
<!-- Vue2 scoped 深度选择器 -->
<style scoped lang="less">
.parent /deep/ .child {
  color: red;
}
.parent >>> .child {
  color: red;
}
::v-deep .child {
  color: red;
}
</style>
```

```vue
<!-- Vue3 scoped 深度选择器 -->
<style scoped lang="less">
.parent :deep(.child) {
  color: red;
}
</style>
```

---

## 2. 插槽选择器迁移

### 2.1 选择器对照

| Vue2 | Vue3 |
|------|------|
| `::v-slotted(.content)` | `:slotted(.content)` |

### 2.2 迁移示例

```vue
<!-- Vue2 -->
<style scoped>
::v-slotted(.slot-content) {
  border: 1px solid #ccc;
}
</style>

<!-- Vue3 -->
<style scoped>
:slotted(.slot-content) {
  border: 1px solid #ccc;
}
</style>
```

---

## 3. 全局选择器迁移

### 3.1 选择器对照

| Vue2 | Vue3 |
|------|------|
| `::v-global(.global-class)` | `:global(.global-class)` |

### 3.2 迁移示例

```vue
<!-- Vue2 -->
<style scoped>
::v-global(.global-class) {
  font-size: 14px;
}
</style>

<!-- Vue3 -->
<style scoped>
:global(.global-class) {
  font-size: 14px;
}
</style>
```

### 3.3 混用全局与局部样式

```vue
<style scoped>
/* 局部样式 */
.container { padding: 20px; }

/* 覆盖子组件样式 */
:deep(.el-input__inner) {
  border-color: #409eff;
}

/* 全局样式 */
:global(.app-wrapper) {
  min-height: 100vh;
}
</style>
```

---

## 4. 过渡动画类名更新

### 4.1 类名对照

| Vue2 | Vue3 | 说明 |
|------|------|------|
| `.fade-enter` | `.fade-enter-from` | 进入动画起点 |
| `.fade-leave` | `.fade-leave-from` | 离开动画起点 |
| `.fade-enter-active` | `.fade-enter-active` | 不变 |
| `.fade-leave-active` | `.fade-leave-active` | 不变 |
| `.fade-enter-to` | `.fade-enter-to` | 不变 |
| `.fade-leave-to` | `.fade-leave-to` | 不变 |

### 4.2 迁移示例

```less
/* Vue2 */
.fade-enter, .fade-leave {
  opacity: 0;
}
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}

/* Vue3 */
.fade-enter-from, .fade-leave-from {
  opacity: 0;
}
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}
```

---

## 5. Scoped 样式适配注意事项

### 5.1 组件根元素样式

Vue3 中 scoped 样式仍然可以作用于组件根元素，行为与 Vue2 一致，无需特殊处理。

### 5.2 子组件根元素样式穿透

```vue
<!-- 父组件覆盖子组件根元素样式 -->
<style scoped>
/* Vue2：scoped 会自动作用于子组件根元素 */
.card { margin: 10px; }

/* 如果需要穿透到深层子组件 */
:deep(.inner-class) {
  color: red;
}
</style>
```

### 5.3 动态 CSS（v-bind in CSS）

Vue3 支持在 `<style>` 中使用 `v-bind()` 绑定响应式数据：

```vue
<script setup>
const themeColor = ref('#409eff')
</script>

<style scoped>
.button {
  background-color: v-bind(themeColor);
}
</style>
```

---

## 6. Less 样式文件迁移要点

### 6.1 变量定义与使用（保持不变）

```less
// Vue2 & Vue3（Less 变量语法不变）
@primary-color: #409eff;
@font-size-base: 14px;

.container {
  color: @primary-color;
  font-size: @font-size-base;
}
```

### 6.2 混合（Mixin）语法（保持不变）

```less
// Vue2 & Vue3（Less mixin 语法不变）
.flex-center() {
  display: flex;
  align-items: center;
  justify-content: center;
}

.header {
  .flex-center();
}
```

### 6.3 嵌套规则（保持不变）

```less
// Vue2 & Vue3（Less 嵌套语法不变）
.card {
  &__header { font-weight: bold; }
  &__body { padding: 16px; }
}
```

---

## 7. CSS 变量（Custom Properties）适配

### 7.1 全局 CSS 变量定义

```css
/* Vue2 */
:root {
  --primary: #409eff;
  --text-color: #333;
}

/* Vue3（语法不变，但建议迁移为更模块化方式） */
:root {
  --primary: #409eff;
  --text-color: #333;
}
```

### 7.2 组件内动态 CSS 变量

```vue
<!-- Vue3 使用 v-bind 在 CSS 中 -->
<style scoped>
.title {
  color: v-bind(titleColor);
}
</style>
```

---

## 8. 组件库样式覆盖

### 8.1 全局样式覆盖

```less
// Vue2
.el-button--primary {
  background-color: @primary-color;
}

// Vue3（Element Plus 类名与 Element UI 基本一致）
.el-button--primary {
  background-color: @primary-color;
}
```

### 8.2 组件内样式覆盖

```vue
<!-- 使用 :deep() 覆盖组件库内部样式 -->
<style scoped lang="less">
.my-form {
  :deep(.el-form-item__label) {
    color: #666;
    font-weight: 500;
  }
  :deep(.el-input__inner) {
    border-radius: 4px;
    &:focus {
      border-color: @primary-color;
    }
  }
}
</style>
```

---

## 9. 迁移检查清单

迁移样式文件时，需检查以下内容是否已处理：

- [ ] `/deep/` → `:deep()` 替换
- [ ] `>>>` → `:deep()` 替换
- [ ] `::v-deep` → `:deep()` 替换
- [ ] `::v-slotted` → `:slotted()` 替换
- [ ] `::v-global` → `:global()` 替换
- [ ] `.v-enter` → `.v-enter-from` 过渡类名更新
- [ ] `.v-leave` → `.v-leave-from` 过渡类名更新
- [ ] Less 变量语法确认无误
- [ ] Scoped 属性保留确认