## 我们先来看看[Bug现象](http://yzc.test/index.php?route=/account/customerpartner/buyergroup/addgroup&type=add)是什么样的

<v-click>

主要表现为以下现象

</v-click>

<v-clicks>

- 切换了类型后，红色报错提示的样式转移到了另一个元素的位置。反复切换类型后，红色报错提示的样式的位置来回变化
- 位置变化处组件定义的校验规则没有触发，表单直接校验通过

</v-clicks>

<v-click>

知道了这个bug有哪些表现，接下来我们来根据现象一条条分析

</v-click>
