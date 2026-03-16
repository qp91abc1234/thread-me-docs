## 特性
### 缓存
微信小程序的缓存和内嵌 webview 的缓存不是一个地方

### 样式隔离 styleIsolation
小程序原生：isolated 表示启用样式隔离，在自定义组件内外，使用 class 指定的样式将不会相互影响
uniapp：apply-shared 表示页面 wxss 样式将影响到自定义组件，但自定义组件 wxss 中指定的样式不会影响页面

## 组件
### image
h5 环境不支持懒加载，小程序环境支持

### 组件封装
在 uni-app 基于组件进行二次封装（尤其面向小程序）时，ex.wd-img -> lazyImg
1.由于组件节点的存在，无法像 web 开发一样将组件外的属性透传到组件根节点
2.`v-bind="$attrs"` 的写法会导致报错 `v-bind="" is not supported`
因此基于组件进行二次封装时，需要显示定义 props