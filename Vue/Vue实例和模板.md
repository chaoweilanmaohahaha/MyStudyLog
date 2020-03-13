# Vue实例以及模板

## 创建一个Vue实例

每一个Vue应用都是由一个Vue实例开始的：

```
var vm = new Vue({
  
})
```

但是如果说我是用npm安装了vue应用，在main.js中只有一个没有赋值的Vue实例：

```javascript
new Vue({
  el: '#app',
  router,
  components: { App },
  template: '<App/>'
})
```

在这个实例中我们能够发现它相当于注册了一个根组件，而这个根组件通过import 方法导入进来：

```
import App from './App'
```

最终这个组件的内容作用在了项目中存在的一个index.html上，当我们运行这个vue应用时，会发现凡是在这个App组件中出现的页面内容都被反映在了index.html上。因此实际后面的开发建立在如何编写这个Vue中注册的组件，然后拼凑出一个正确的页面。

## 模板

官网上是这么说的：

> 在底层的实现上，Vue 将模板编译成虚拟 DOM 渲染函数。结合响应系统，Vue 能够智能地计算出最少需要重新渲染多少组件，并把 DOM 操作次数减到最少。

