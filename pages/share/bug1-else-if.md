## 那么还有其他更简单合理的方案吗

观察两组列表，可以发现其中有非常多的注释类型的vnode，这里我们排除掉那些不会影响`patch`时vnode顺序的节点便于观察

```ts {monaco-diff}{lines: true}
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

---

````md magic-move {lines: true}
```ts
// 旧vnode列表
const vnode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select2' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined }, // 清除这个多余的注释节点
];
```

```ts
// 旧vnode列表
const vnode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select2' }, // 我俩可以patch了
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
];
```


````

````md magic-move {lines: true}
```ts
// 新vnode列表
const vnode = [
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined }, // 清除这个多余的注释节点
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select2' },
];
```
```ts
// 新vnode列表
const vnode = [
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined }, // 我俩可以patch了
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select2' },
];
```
````

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

---

```javascript {all|4-17}
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

---

在vue2中，渲染函数中所使用的`helper`辅助函数都通过别名挂在到`_vm`实例中，例如此处的`_v`和`_e`方法，挂载的方法如下

```ts {all|13-14}{maxHeight:'300px'}
export function installRenderHelpers(target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
  target._d = bindDynamicKeys
  target._p = prependModifier
}
```

找到`_v`和`_e`对应的方法，可以看到分别是`createTextVNode`和`createEmptyVNode`的方法别名。很容易从函数名看出一个是创建文本vnode，一个是创建空的vnode。

---

```javascript {all|4-17}
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

也就是说，上面的渲染函数中`_setup.boolean`部分对应的`v-if`的编译结果是，条件结果为真时正常渲染结果，为否时渲染空的注释节点，这也就是为什么上面的vnode列表中有很多空节点，因为同时只会有一个表达式为true，所以始终会有一个三元表达式的结果是一个空的注释节点

所以既然`v-if`的渲染存在空注释节点的问题，我们需要去查看渲染函数的生成，看看`v-if`的模板是如何编译成这样的渲染函数的

---
layout: two-cols
layoutClass: gap-16
---

```ts
// src\compiler\parser\index.ts
// parser阶段

// 记录下模板定义的v-if判断条件
function addIfCondition(el: ASTElement, condition: ASTIfCondition) {
  if (!el.ifConditions) {
    el.ifConditions = []
  }
  // 保存在ifConditions字段中
  el.ifConditions.push(condition)
}

function processIf(el) {
  // 获取指定attribute的值
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    el.if = exp
    // 记录v-if表达式
    addIfCondition(el, {
      exp: exp,
      block: el
    })
  } else {
    if (getAndRemoveAttr(el, 'v-else') != null) {
      el.else = true
    }
    const elseif = getAndRemoveAttr(el, 'v-else-if')
    if (elseif) {
      el.elseif = elseif
    }
  }
}
```

::right::

```ts
// src\compiler\codegen\index.ts
// generatecode阶段

function genIf(
  el: any,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  el.ifProcessed = true // avoid recursion
  return genIfConditions(el.ifConditions.slice(), state, altGen, altEmpty)
}

function genIfConditions(
  conditions: ASTIfConditions,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  if (!conditions.length) {
    // 没有条件了 生成空的注释节点
    return altEmpty || '_e()'
  }

  // 从记录的ifConditions头部取出条件表达式
  const condition = conditions.shift()!
  if (condition.exp) {
    // 生成三元表达式
    return `(${condition.exp})?${genTernaryExp(
      condition.block
    )}:${genIfConditions(conditions, state, altGen, altEmpty)}`
  } else {
    return `${genTernaryExp(condition.block)}`
  }
}
```

可以看到单独的`v-if`由于只有一个条件，在第二次调用`genIfConditions`的时候条件已经为空了，所以生成了空节点，那么如果只要这时候条件不为空，就可以避免影响`patch`的注释节点产生

---

这时候我们再看项目中代码

```vue {maxHeight:'100px'}
<!-- 多选下拉 -->
<template v-if="item.input_type === 'select_multi'">
  <div class="w-20%">
  </div>
  <div class="w-40%">
  </div>
</template>
<!-- 单选 -->
<template v-if="item.input_type === 'select_single'">
  <template v-if="item.type === 'boolean'">
    <div class="w-20%">
    </div>
    <div class="w-40%" />
  </template>
  <template v-else>
    <div class="w-20%">
    </div>
    <div class="w-40%">
    </div>
  </template>
</template>
<!-- 主营品类 -->
<template v-if="item.input_type === 'select_category'">
  <div class="w-20%">
  </div>
  <div class="w-40%">
  </div>
</template>
<!-- 输入框 区间 -->
<template v-if="item.input_type === 'int_range' || item.input_type === 'float_range'">
  <div class="w-40%">
    <div class=" mr-12px flex items-center">
    </div>
  </div>
  <div class="w-20%" />
</template>
```

可以看到每个`template`部分其实都是渲染同一个地方，且`v-if`条件其实本身就存在互斥的关系，因此此处应该在第一个`v-if`之后改为使用`v-else-if`进行判断，这样写的作用是会把后续的判断条件放到`ifConditions`数组中，在生成渲染函数的时候便不会直接去生成多个单独的包含空注释节点的三元表达式，而是放在一个嵌套的三元表达式中

---

还是用刚才的简单的demo举例，改为`v-else-if`后，生成的渲染函数变为

```javascript
function render() {
  var _vm = this, _c = _vm._self._c, _setup = _vm._self._setupProxy;
  return _c("div", { staticClass: "demo" }, 
    [_setup.boolean 
      ? [_c("div", { ref: "div1", staticStyle: { "color": "orange" } }, [_vm._v(" 我是直的 ")]), _c("div", { staticStyle: { "color": "red" } }, [_vm._v(" 我是红色 ")])] 
      : _setup.boolean === false
       ? [_c("div", { staticStyle: { "color": "blue" } }, [_vm._v(" 我是蓝色 ")]), _c("div", { ref: "div4", staticStyle: { "color": "green" } }, [_vm._v(" 我是瘦的 ")])]
       : _vm._e()], 2);
}
```

可以看到两处`_setup.boolean`的判断条件都并到了一个三元表达式中，这样在条件发生切换时，都只会影响一条三元表达式的值，而不会出现多余的注释空节点影响`patch`的结果。

同样的，我们可以把出问题的代码中多的`v-if`改为`v-else-if`，发现问题也同样解决了

---
