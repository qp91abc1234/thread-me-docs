## 模块图生成（伪代码）

```js
// 入口
buildModuleGraph('/src/main.js')

async function buildModuleGraph(entryId) {
  const moduleGraph = new Map()

  async function processModule(importee, importer = null) {
    // 1. resolveId：把 import 路径解析成唯一 id
    const id = await resolveId(importee, importer)
    // './App.vue' -> '/src/App.vue'

    // 避免重复处理
    if (moduleGraph.has(id)) {
      return moduleGraph.get(id)
    }

    // 创建模块节点
    const moduleNode = {
      id,
      code: '',
      deps: []
    }

    moduleGraph.set(id, moduleNode)

    // 2. load：加载模块源码
    let code = await load(id)
    // '/src/App.vue' -> '<template>...</template>'

    // 3. transform：转换源码
    code = await transform(code, id)
    // App.vue -> import '/src/App.vue?vue&type=style'

    moduleNode.code = code

    // 4. 分析 transform 后的 import
    const imports = parseImports(code)

    // 5. 递归处理依赖
    for (const childImportee of imports) {
      const childModule = await processModule(childImportee, id)
      moduleNode.deps.push(childModule.id)
    }

    return moduleNode
  }

  await processModule(entryId)

  return moduleGraph
}
```

## hmr

### 开发服务器侧
```
监听文件变化
↓
找到受影响模块
↓
通过 websocket 通知浏览器
```

### 浏览器侧（HMR Runtime）
```
接收 websocket 更新通知
↓
重新请求变更后的模块
↓
根据模块 id 找到对应的 hot.accept 回调
↓
执行热更新逻辑
↓
更新组件实例
↓
重新 render
↓
patch DOM
```

### 伪代码

```js
const hotModulesMap = new Map()

function createHotContext(moduleId) {
  return {
    accept(callback) {
      hotModulesMap.set(moduleId, {
        accepted: true,
        callback
      })
    }
  }
}

import.meta.hot = createHotContext('/src/App.vue')
```

```js
// App.vue transform 后
const component = {
  render() {
    return 'hello'
  }
}

export default component

// 注册 HMR
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // 用新 render 更新旧组件
    rerender(component, newModule.default)
  })
}
```

```js
socket.onmessage = async (msg)=>{
  handleHMRUpdate(msg.path)
}

async function handleHMRUpdate(path) {
  // 重新请求新模块
  const newModule = await import(
    path + '?t=' + Date.now()
  )

  // 找到旧模块注册的信息
  const oldModule =
    hotModulesMap.get(path)

  // 当前模块接受热更新
  if (oldModule.accepted) {
    // 执行 accept 回调
    oldModule.callback(newModule)

  } else {
    // 当前模块不会处理热更新
    // 往父模块传播
    propagateUpdate(path)
  }
}
```

```js
function rerender(oldComp, newComp){
  // 更新组件实例
  oldComp.render = newComp.render

  // 找到所有组件实例
  const instances =
    getInstances(oldComp)

  // 重新 render
  const newVNode =
    oldComp.type.render()

  // patch DOM
  patch(oldVNode, newVNode)
}
```