## 问题1

### 切换了类型后，红色报错提示的样式转移到了另一个元素的位置。反复切换类型后，红色报错提示的样式的位置来回变化

我们可以发现这个问题的重点是报错元素的 **位置** 在数据更新后发生了错误的变化。

<v-clicks>

如果是在使用jQuery这种命令式库的项目中产生这种问题，解决起来就会非常清晰。因为每个DOM元素的位置和顺序等都是由用户自己手动操作实现，只需要查看每个DOM操作步骤即可发现是哪一步产生了错误

而在Vue中则不同，Vue属于声明式框架，我们在开发时编写内容的关注点是 **"要什么"**。我们描述的是期望达到的最终状态，而不是达到这个状态的具体步骤。

如何理解呢，举个简单的例子，有一个按钮，初始显示“显示内容”。点击按钮，按钮文本变为“隐藏内容”，同时一个隐藏的`div`元素显示出来。再次点击，按钮文本变回“显示内容”，`div`再次隐藏。那么使用jQuery的话，代码可能是这样

</v-clicks>

---

````md magic-move
```html
<style>
  #hiddenContent {
      display: none; /* 初始隐藏 */
  }
</style>
<button id="toggleButton">显示内容</button>
<div id="hiddenContent">
    <p>这是隐藏的内容。</p>
    <p>它只有在按钮被点击时才显示。</p>
</div>
<script>
  const $button = $('#toggleButton');
  const $content = $('#hiddenContent');
  $button.on('click', function() {
      if ($content.is(':hidden')) { // 检查当前状态（从DOM推断）
          // 命令式地修改DOM
          $content.show(); // 显示div
          $button.text('隐藏内容'); // 修改按钮文本
      } else {
          // 命令式地修改DOM
          $content.hide(); // 隐藏div
          $button.text('显示内容'); // 修改按钮文本
      }
  });
</script>

```

```vue
<template>
  <div>
    <button @click="isVisible = !isVisible" :class="{ 'active-button': isVisible }">
        {{ isVisible ? '隐藏内容' : '显示内容' }}
    </button>
    <div v-show="isVisible" class="content-div">
        <p>这是隐藏的内容。</p>
        <p>它只有在按钮被点击时才显示。</p>
    </div>
  </div>
</template>
<script>
  export default {
      data: {
          isVisible: false
      }
  };
</script>
```
````

<v-click>

我们只需要写好结构，至于事件和修改样式等操作，则由底层系统（在前端框架中就是框架本身）来完成，所以类似于在jQuery中我们在数据变化时手动操作DOM的行为，大部分情况下都已经被Vue接管了，我们没有对DOM元素本身进行操作，我们只是修改了数据，页面的变化是由Vue来完成的

因此要探究元素位置的错误，我们需要去了解Vue在数据变化时是如何反映到真实元素上的。

</v-click>


---

在Vue中，框架去接管的真实元素的渲染部分就是我们熟悉的基于vNode（虚拟DOM）这个概念构建的渲染机制，编译时把模板编译成渲染函数，即运行时生成vNode树，通过渲染器遍历vNode树，根据情况决定是挂载(`mount`)还是更新(`patch`)，自动地完成页面的渲染过程

<n-image src="./share/render.png" class="h-300px" />

<v-click>

图中可以看到，从vNode树到真实的Dom，中间经过了一个mount/patch的过程，具体的对于真实元素操作发生的时机是在这张渲染流程示意图中的`patch`部分，所以我们需要去了解一下`patch`过程中具体是如何影响到元素的位置变化的

</v-click>

---


1. `patch`是什么，为什么需要`patch`

在 Vue 中，`patch` (打补丁) 是一个核心概念，它的主要职责是：比较新旧两棵 vNode 树（虚拟 DOM 树），然后将比较结果反映到真实的 DOM 上，以 **最少** 的 DOM 操作来更新视图。可以简单理解为：

`patch(oldVnode, newVnode)` = 计算差异 (`diff`) + 应用差异 (`apply`) 到真实 DOM。

<v-clicks>

那么为什么需要这个步骤呢

假如我们有一个旧的虚拟 DOM 树 (oldVnode)，它对应着当前浏览器中显示的真实 DOM。同时有一个新的虚拟 DOM 树 (newVnode)，它代表了数据改变后视图应该有的样子。如果没有`patch`这个步骤，我们需要简单的去把当前页面对应的真实 DOM 删除，然后把新的vNode树渲染成真实 DOM 插入到页面中完成渲染，这样做的后果是什么呢

1. 因为只是无脑把 DOM 销毁再重建，即使只是改了一个文案，也需要把整个 DOM 树重新渲染出来，性能存在问题

2. 用户的输入框失焦，滚动条位置变化，动画效果消失等等，体验不好

因此，我们需要一个`patch`的过程，利用虚拟 DOM 的优势，通过高效的算法来尽量优化真实 DOM 的操作，例如复用 DOM 元素等等。

</v-clicks>

---

2. `vNode`和`sameVnode`

`vNode`就是虚拟节点，是一个包含描述真实元素属性的 **普通** 的`js`对象，想构造一个vNode其实非常简单，只需要描述出一个节点的重要属性即可，例如

```js
// 描述了一个div节点，内容为一个DIV
const vNode = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一个DIV'
}
```

这样就得到了一个简单的vNode，Vue中的vNode的结构就类似这样，只是会添加上很多附属的字段用于详细描述节点的各个属性。

`patch`阶段所谓的优化真实 DOM 操作的过程其实就是获取到数据变化前后的vNode树进行对比，找出差异点在运用到对应的DOM上

<!--
1. 为什么需要虚拟节点 A:在声明式框架下的平衡方案
2. 为什么需要sameVnode，意义是什么 A:新旧vNode描述的内容需要一致，例如旧vNode渲染一个p元素，新的vNode渲染一个img元素，这样就失去了打补丁的意义，因为我们要先卸载旧的元素新建新的元素才能实现效果，无法在旧的元素上通过添加属性等操作来实现
-->

---

而`sameVnode`的概念，则表示两个vNode节点描述的是同种节点。为什么要强调同种节点呢？试想一下，假设我们有一对新旧节点如下

```ts
const oldVnode = {
  tag: 'div',
  text: '一个div',
  data:{
    class: 'divClass'
  }
}

const newVnode = {
  tag: 'span',
  text: '一个span',
  data: {
    class: 'spanClass'
  }
}
```

如果我们要对其进行复用的话，即使把旧的节点对应的真实元素的`text`和`class`都替换成新的节点对应的值，我们的操作仍然是对原来的`div`进行修改，不管怎么修改他都不会变成`span`，不符合vNode描述的真实DOM元素的结构。因为这两个节点类型就是不同的，不能走简单的复用逻辑，而是需要走新建和卸载的流程。

---

所以我们需要`sameVnode`的原因就是为了确认前后两个vNode是否能走`patch`复用修补的逻辑，满足`sameVnode`的条件，才有接下来`patch`的过程。

在Vue中，有专门的方法来判断两个vNode是否是`sameVnode`的方法

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

满足这些条件则视为同种vNode，这也是Vue中两个vNode可以进行`patch`的前提条件

---

例如

<div class="flex gap-12px">
<div class="flex-1">

```js
// 我们是sameVnode 因为tag key相同
const vNode1 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一号DIV'
}

const vNode2 = {
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
const vNode1 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一号DIV'
}

const vNode2 = {
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
const vNode1 = {
  tag: 'div',
  key: '1',
  data: {},
  text: '一号DIV'
}

const vNode2 = {
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

<n-image src="./share/patch流程.png" alt="" />

::right::

这是一个简略的`patch`过程的描述示意图

<v-click>

观察`patch`的过程可以发现，修补vNode的过程实际是在`patchVnode`方法中，且在`updateChildren`处理之前几乎都是对于单个节点的修补过程，例如如果是组件，那就更新一下组件上的传值、事件等等，对应的真实元素上的一些`class`之类的属性，并 **没有** 涉及到元素之间的顺序等操作，之后便是递归地去处理子节点列表的过程。真正涉及到真实元素顺序变化的地方是在`updateChildren`方法里，所以我们需要查看`updateChildren`的内部工作原理来找出影响元素位置的原因。

</v-click>

---

`updateChildren`方法就是大家非常熟悉的双端对比`diff`算法，简单来说就是按照一定的对比顺序对比新旧vNode的子节点列表，尽可能找出可以复用的vNode，之后重复走`patchVnode`的流程，两个两个节点进行复用和修补的过程。

<v-clicks>

双端对比的顺序简单概括起来就是

1. 头头比较
2. 尾尾比较
3. 旧头与新尾比较（如果可以`patch`，在处理后需要进行元素顺序调整）
4. 旧尾与新头比较（如果可以`patch`，在处理后需要进行元素顺序调整）
5. 构建`key`和`index`的map对象，根据新头的`key`去查找是否有可`patch`的节点（如果可以`patch`，在处理后需要进行元素顺序调整）
6. 没有可`patch`节点，新建真实元素

</v-clicks>

<v-click>

所以此时需要观察有bug的页面中选项切换前后的vNode列表的`diff`过程，看看是否存在上述几种有位置移动的`diff`场景存在。

</v-click>

---

### 查看选项切换前后vNode列表

```ts {monaco-diff}
// 空白占位vNode的isComment属性为true
const vNode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select1' },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select2' },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select3' },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
];
~~~
// 空白占位vNode的isComment属性为true
const vNode = [
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select1' },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-20%" },children:'el-select3' },
  { tag: "div", isComment: false, key: undefined, text: undefined, data: { staticClass: "w-40%" },children:'el-select2' },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
  { tag: undefined, isComment: true, key: undefined, text: "", data: undefined },
];
```

这里我们可以通过一个简单的演示动画直观的查看过程

<v-clicks>

根据演示，可以发现在排除了头部和尾部直接可以`patch`的vNode节点后，旧vNode列表的**头部**和新vNode列表的**尾部**的vNode进行了`patch`，这意味着这个节点被复用的同时，Vue还会把它对应的DOM元素的位置进行调整，将旧的头部节点对应的DOM移动到了新节点的位置，因此位置发生了变化

</v-clicks>

---

## 解决方案

既然知道了原因，那么解决方法就很容易想到了，只要不让两个节点错误的`patch`即可解决这个问题，根据前面提到的`sameVnode`方法的判断条件，这里我们给出两个解决方案：

<v-clicks>

1. 我们可以不让这个节点进行复用，即在`diff`阶段不让其视为同一类节点。最简单的方法就是自然是给vNode添加不同的自定义的`key`，`key`不同自然也就无法进行修补，只能新增对应的节点，这样也就不存在复用过程中DOM顺序变化的问题了。
2. 既然发生了错误的位置的`patch`，我们只要手动给在正确位置`patch`的两个模板节点添加对应的`key`，手动指定一个可以正确复用的节点顺序的key，而不是使用Vue默认的头尾节点对比，同样也可以解决这个问题。

</v-clicks>

<!--
两种解决方案分别在源代码中进行演示，展示效果
-->

---
src: ./bug1-else-if.md
---
