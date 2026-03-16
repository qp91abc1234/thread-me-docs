## 缓存
微信小程序的缓存和内嵌 webview 的缓存不是一个地方

## image
h5 环境不支持懒加载，小程序环境支持

## 样式隔离 styleIsolation
小程序原生：isolated 表示启用样式隔离，在自定义组件内外，使用 class 指定的样式将不会相互影响
uniapp：apply-shared 表示页面 wxss 样式将影响到自定义组件，但自定义组件 wxss 中指定的样式不会影响页面