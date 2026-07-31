## 组件节点

**Vue Web** 下自定义组件在编译后没有多出组件节点这一层；**小程序**等端会在编译后多出这一层组件节点，作为组件的父节点。

该层是一层**真实节点**，如同 `view`，能设 class/style、参与 flex、受继承样式影响。

```
// comp.vue 编译后结构
<pages/index/comp>   ← 组件节点
  #shadow-root
    <view class="comp"></view>
</pages/index/comp>
```

---

## 虚拟化组件节点

虚拟化后，组件节点仍在树中，但该节点：

- **不参与布局、不应用样式**
- 父级写在组件标签上的 class / style / 动画**不生效**，也**不会传到组件内第一层**

**作用**：让组件内第一层节点直接参与父级布局。

> 无法全局开启，只能在组件内通过 `defineOptions` 开启。

**组件内配置：**

```ts
defineOptions({
  options: {
    virtualHost: true
  }
})
```

**编译后结构：**

```
<pages/index/comp>   ← 虚拟化组件节点
  #shadow-root
    <view class="comp"></view>
</pages/index/comp>
```

---

## mergeVirtualHostAttributes

- **uni-app 独有配置**，配合虚拟化组件节点使用。
- 将虚拟化组件节点上的属性**合并到组件根节点**。
- 目前仅支持：`id`（v4.42+）、`style`（v3.5.1+）、`class`（v3.5.1+）、以及 v-show 指令生成的 `hidden`（v4.41+）。
- 开启后，开发效果接近 **Vue Web**，但无法将属性透传到根节点。

**全局配置（manifest.json）：**

```json
{
  "mp-weixin": {
    "mergeVirtualHostAttributes": true
  }
}
```

---

## styleIsolation

样式隔离：微信小程序原生默认为 `isolated`，uni-app 默认为 `apply-shared`。无法全局设置，需在页面/组件内单独配置。

**组件内配置：**

```ts
defineOptions({
  options: {
    styleIsolation: 'apply-shared'
  }
})
```

| 模式 | 页面样式 | 组件样式 | 页面 DOM | 组件 DOM | 效果 |
| --- | --- | --- | --- | --- | --- |
| `isolated` | 带页面后缀 | 带组件后缀 | 带页面属性 | 带组件属性 | 完全隔离 |
| `apply-shared` | 带页面后缀 + 一份无后缀副本 | 带组件后缀 | 带页面属性 | 带组件属性 | 页面样式可影响组件，组件样式不影响页面 |
| `shared` | 带页面后缀 + 一份无后缀副本 | 带组件后缀 + 一份无后缀副本 | 带页面属性 | 带组件属性 | 双向互相影响 |

> `scoped` 是 Vue3 的作用域样式，可在小程序样式隔离之上叠加。例如 `styleIsolation: shared` 时，用 `scoped` 可避免子组件样式影响父组件。
