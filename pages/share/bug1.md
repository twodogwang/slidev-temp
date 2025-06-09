## 问题1：

### 切换了类型后，红色报错提示的样式转移到了另一个元素的位置。反复切换类型后，红色报错提示的样式的位置来回变化

这个问题的重点是某个元素的**位置**发生了变化。

<v-clicks depth="2">

- 如果是在使用jQuery的项目中产生这种问题，解决起来就会非常清晰。因为每个dom元素的位置和顺序等都是由用户自己手动操作实现，只需要查看每个dom操作步骤即可发现是哪一步产生了错误。
- 而在vue中则不同，vue属于声明式框架，声明式编程关注的是 **"要什么"**。你描述的是期望达到的最终状态，而不是达到这个状态的具体步骤。至于如何实现这个状态，则由底层系统（在前端框架中就是框架本身）来完成，所以类似于在jQuery中手动操作dom的操作，大部分情况下都已经被vue接管了，大家可以回忆一下，除去自己手动去操作dom元素顺序的情况，一般来说，元素的位置顺序发生变化的场景是什么？

  - 例如渲染一个数组类型的数据，数据本身的顺序发生变化，对应的dom元素位置也发生变化

</v-clicks>

---

在vue中，框架去接管的真实元素的渲染部分就是我们熟悉的基于vnode这个概念构建的渲染机制，即运行时渲染器遍历vnode树，根据情况决定是挂载(`mount`)还是更新(`patch`)，使用者不需要手动去执行对应的真实元素的操作，具体的操作交给vue的渲染器去处理

<n-image v-click src="/share/render.png" />

<v-click>

具体的对于真实元素操作发生的时机是在这张渲染流程示意图中的`patch`部分，所以我们需要去了解一下`patch`过程中具体是如何影响到元素的位置变化的

</v-click>

---

了解`patch`过程之前，我们需要知道两个概念，即`vNode`和`sameVnode`

`vnode`就是虚拟节点，是一个包含描述真实元素属性的普通的`js`对象，例如

```js
// 描述了一个div节点，内容为一个DIV
const vnode = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一个DIV'
}
```

这样就得到了一个简单的vnode，vue中的vnode的结构就类似这样，只是会添加上很多附属的字段用于详细描述节点的各个属性，同时它也是渲染函数的返回值，`patch`阶段就是其实针对它进行处理

<!--
1. 为什么需要虚拟节点 A:在声明式框架下的平衡方案
2. 为什么需要sameVnode，意义是什么 A:新旧vnode描述的内容需要一致，例如旧vnode渲染一个p元素，新的vnode渲染一个img元素，这样就失去了打补丁的意义，因为我们要先卸载旧的元素新建新的元素才能实现效果，无法在旧的元素上通过添加属性等操作来实现
-->

---

在vue中，有专门的判断是否是`sameVnode`的方法

```ts
function sameVnode(a, b) {
  return (
    // key相同 异步组件相关逻辑
    a.key === b.key && a.asyncFactory === b.asyncFactory
    // tag相同 是否是注释节点
    && ((a.tag === b.tag && a.isComment === b.isComment 
    // 是否定义了data
    // 如果是input类型特殊处理 是否是同种input类型等逻辑处理
    && isDef(a.data) === isDef(b.data) && sameInputType(a, b)) 
    // 异步占位节点相关逻辑
    || (isTrue(a.isAsyncPlaceholder) && isUndef(b.asyncFactory.error)))
  )
}
```

满足这些条件则视为同种vnode，这也是两个vnode可以进行`patch`的必要条件

---

例如

<div class="flex gap-12px">
<div class="flex-1">

```js
// 我们是sameVnode 因为tag key相同
const vnode1 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一号DIV'
}

const vnode2 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '二号DIV'
}

```

</div>
<div class="flex-1">

```js
// 我们不是sameVnode 因为tag不同
const vnode1 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一号DIV'
}

const vnode2 = {
  tag: 'span',
  key: '1',
  data: {},
  text: '一号SPAN'
}
```

</div>
<div class="flex-1">

```js
// 我们不是sameVnode 因为key不同
const vnode1 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一号DIV'
}

const vnode2 = {
  tag: 'div',
  key: '2',
  data: {},
  text: '二号DIV'
}
```

</div>
</div>

---
layout: two-cols
layoutClass: gap-16
---

<n-image src="/share/patch流程.png" alt="" />

::right::

<v-click>

观察`patch`的过程可以发现，修补vnode的过程实际是在`patchVnode`方法中，且在`updateChildren`处理之前都是真实元素的复用和属性处理，真正涉及到真实元素顺序变化的地方只有`updateChildren`方法，所以只需要查看`updateChildren`的内部工作原理即可找出该bug影响元素位置的原因。

</v-click>

---

`updateChildren`方法即大家非常熟悉的双端对比diff法，简单来说就是按照一定的对比顺序对比新旧vnode的子节点列表，尽可能找出可以复用的vnode，之后重复走`patchVnode`的流程，也就是递归地两个两个节点进行`patch`。


<v-clicks>

双端对比的顺序概括起来就是

1. 头头比较
2. 尾尾比较
3. 旧头与新尾比较（位置移动）
4. 旧尾与新头比较（位置移动）
5. 构建`key`和`index`的map对象，根据新头的key去查找是否有可`patch`的节点（位置移动）
6. 没有可`patch`节点，新建真实元素

</v-clicks>

<v-click>

所以此时需要观察有bug的页面中选项切换前后的vnode列表的`patch`过程，看看是否存在上述几种有位置移动的diff场景存在。

</v-click>

---

### 查看选项切换前后vnode列表

```ts {monaco-diff}{lines: true}
// 空白占位vnode的isComment属性为true
const vnode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select1' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select2' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
];
~~~
// 空白占位vnode的isComment属性为true
const vnode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select1' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select3' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select2' },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: false, key: undefined, text: " ", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
];
```

根据刚才的双端对比法，我们可以发现在排除了头部和尾部直接可以`patch`的vnode节点后，旧vnode列表的头部和新vnode列表的尾部的vnode可以进行`patch`，这意味着这个被复用的节点位置发生了变化，也就解释了为什么出现了这个元素位置反复变化的现象，其实就是简单的vnode复用的问题。

---

# 解决方案

既然知道了原因，那么解决方法就很容易想到了，只要不让两个节点错误的`patch`即可解决这个问题，这里我们给出两个解决方案：

<v-clicks>

1. 比如我们可以不让这个节点进行复用，即在patch阶段不让其视为同一类节点。最简单的方法就是自然是给vnode添加不同的自定义的`key`，`key`不同自然也就无法进行修补，只能新增对应的节点，这样也就不存在复用过程中dom顺序变化的问题了。
2. 或者我们手动给将要`patch`的两个模板节点添加对应的`key`，手动指定一个可以正确复用的节点顺序的key，而不是使用vue默认的头尾节点对比，同样也可以解决这个问题。

</v-clicks>

<!--
两种解决方案分别在源代码中进行演示，展示效果
-->

---
src: ./bug1-else-if.md
---
