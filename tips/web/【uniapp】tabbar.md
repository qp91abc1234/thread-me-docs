## 生命周期行为

- **首次进入 tab 页面**：会创建页面实例（触发 `onLoad` -> `onShow`）。
- **切换 tab 时**：页面通常只会触发 `onHide` / `onShow`。
- **页面不会被销毁**：行为类似 `keep-alive`，再次切回通常不走 `onLoad`。

## tabbar 跳转

tabBar 页面只能通过 `switchTab` 进入：

```js
wx.switchTab({ url: '/pages/home/index' })
```

以下方式不支持跳转到 tabBar 页面：

- `navigateTo`
- `redirectTo`

`switchTab` 的核心行为：

- 切换到目标 tabBar 页面；
- 清空当前栈内所有**非 tabBar 页面**；
- 如果目标 tab 已存在，则触发 `onShow`，不会重新触发 `onLoad`。

`switchTab` **不能携带 query 参数**。

错误示例：

```js
wx.switchTab({
  url: '/pages/home/index?id=1'
})
```

推荐方案：

- 使用全局状态（`store` / `storage`）；
- 或使用事件通信。

## 自定义 tabbar 配置

```json
// pages.json
{
  "tabBar": {
    "custom": true,
    "list": [
      {
        "pagePath": "page/component/index",
        "text": "组件"
      },
      {
        "pagePath": "page/API/index",
        "text": "接口"
      }
    ]
  }
}
```

## 首页判定

- `tabBar.list` 的第一项**不是**首页判定依据；
- 小程序首页始终由 `pages.json` 中 `pages` 数组的第一项决定。

示例：

```json
{
  "pages": [
    "pages/index/index",
    "pages/login/index"
  ],
  "tabBar": {
    "list": [
      { "pagePath": "pages/message/index", "text": "消息" },
      { "pagePath": "pages/mine/index", "text": "我的" }
    ]
  }
}
```

上例中，首页是 `pages/index/index`，而不是 `tabBar.list[0]` 的 `pages/message/index`。