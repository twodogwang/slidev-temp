## 那么还有其他更简单合理的方案吗

观察两组列表，可以发现其中有非常多的注释类型的vnode，这里我们排除掉那些不会影响`patch`时vnode顺序的节点便于观察

```ts {monaco-diff}
// 空白占位vnode的isComment属性为true
const vnode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select2' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
];
~~~
// 空白占位vnode的isComment属性为true
const vnode = [
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select2' },
];
```

<v-click>

可以发现两组vnode列表中有许多空的注释节点，例如左侧的最后一个节点和右侧的第一个节点，正是这些注释类型的vnode导致了`patch`过程中位置的变化，因为头部尾部都不再能够直接修补，如果没有这些注释节点，`patch`的时候顺序就不会变化了

</v-click>

<v-clicks> 

- 那么我们就要想办法消除这些多余的影响`patch`过程的节点，怎么消除呢
- 首先我们要了解为什么会多出这些多余的空白注释vnode节点

</v-clicks>

---


<n-space>
  我们来看一个简化版的<span class="underline cursor-pointer" @click="showDrawer">demo</span>

  <n-drawer title="iframe" :active="show" @close="closeDrawer">
    <iframe width="100%" height="100%" :src="url" frameborder="0"></iframe>
  </n-drawer>
</n-space>




<script setup>
  import { ref } from 'vue'
  const show = ref(false)
  const url = ref('')

  function showDrawer() {
    show.value = true
    url.value = 'http://localhost:5174'
  }

  function closeDrawer() {
    show.value = false
  }

  const showRender = ref(false)

</script>

<v-click>

可以看到简化版的demo有着同样的问题，切换前后渲染的dom位置不停变化，现象相同，所以我们可以通过观察这个简化组件来解决上面的问题

</v-click>


<div v-click="2" @click="showRender = true">

```vue
<template v-if="boolean">
  <div ref="div1" style="color: orange;">
    我是直的
  </div>
  <div style="color: red;">
    我是红色
  </div>
</template>
<template v-if="!boolean">
  <div style="color: blue;">
    我是蓝色
  </div>
  <div ref="div4" style="color: green;">
    我是瘦的
  </div>
</template>
<el-button @click="boolean = !boolean">
  切换
</el-button>
```

</div v-click>


<n-drawer :active="showRender" @close="showRender = false" title="渲染函数">

```javascript {4-17|all}
function render() {
  var _vm = this, _c = _vm._self._c, _setup = _vm._self._setupProxy;
  return _c("div", { staticClass: "demo" }, [
    _setup.boolean 
    ? [
      _c("div", { ref: "div1", staticStyle: { "color": "orange" } },
       [_vm._v(" 我是直的 ")]),
        _c("div", { staticStyle: { "color": "red" } },
         [_vm._v(" 我是红色 ")])] 
         : _vm._e(), 
         !_setup.boolean 
         ? [
          _c("div", { staticStyle: { "color": "blue" } },
           [_vm._v(" 我是蓝色 ")]),
            _c("div", { ref: "div4", staticStyle: { "color": "green" } },
             [_vm._v(" 我是瘦的 ")])]
          : _vm._e()], 2);
}

```


<div>

需要重点关注的是渲染函数中`_setup.boolean`这个判断条件后面的渲染内容，
可以看到两个判断条件对应了两个三元运算表达式，
而表达式的结果中除了正常的`v-if`条件渲染的内容，都包含`_vm.e()`这个方法调用的结果。

</div>

</n-drawer>
