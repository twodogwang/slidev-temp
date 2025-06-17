## 表单的校验方法失效

第二个问题就相对简单了，因为问题产生的原因可以确定就是发生在`form`组件中，我们只要从`form`组件的源码入手即可。`element-ui`的表单校验大家都很熟悉了，主要流程是利用`formItem`组件先在template中定义好需要校验的数据字段值`prop`还有校验的方法`rules`，最后通过执行`form`组件的`validate`方法，触发校验流程，返回校验结果。所以我们从校验方法的触发点`validate`开始入手，先来查看它的源码

```javascript {all|24-34}{maxHeight:'222px'}
validate(callback) {
  if (!this.model) {
    console.warn('[Element Warn][Form]model is required for validate to work!');
    return;
  }

  let promise;
  // if no callback, return promise
  if (typeof callback !== 'function' && window.Promise) {
    promise = new window.Promise((resolve, reject) => {
      callback = function(valid) {
        valid ? resolve(valid) : reject(valid);
      };
    });
  }

  let valid = true;
  let count = 0;
  // 如果需要验证的fields为空，调用验证时立刻返回callback
  if (this.fields.length === 0 && callback) {
    callback(true);
  }
  let invalidFields = {};
  this.fields.forEach(field => {
    field.validate('', (message, field) => {
      if (message) {
        valid = false;
      }
      invalidFields = objectAssign({}, invalidFields, field);
      if (typeof callback === 'function' && ++count === this.fields.length) {
        callback(valid, invalidFields);
      }
    });
  });

  if (promise) {
    return promise;
  }
}
```

<v-click>

查看`validate`的方法源码可以发现，内部其实非常简单，除了必要的参数的值的校验和提示信息，主要的校验流程只有调用保存的`fields`数组中各自的`validate`方法而已，所以我们排查的范围缩小到了查看`fields`中保存的内容以及它对应的`validate`方法

</v-click>

---

这里给出一张`form`组件和`formItem`组件在校验实现上的简略流程图。

<n-image src="/share/element-form.png" class="h-350px" />

前面提到的重点，`form`组件的`fields`数组其实保存的就是`formItem`组件，每个`formItem`在`mounted`之后，如果它的`prop`不为空，则会把自身保存在`form`的`fields`中，在`form`校验时，其实就是触发每个`formItem`的校验方法。

---

那么我们自然会考虑到，是否是在切换了选项后，原来的`form`组件的`fields`数组发生了变化，原来可用的`formItem`组件被移除了，导致校验方法失效了呢？

<v-clicks>

<div>
<span>仔细查看<code>form</code>的<code>fields</code>数组中数据的</span>
<Popover width="600">
  <template #trigger>
    <strong class="cursor-pointer"> 变化时机 </strong>
  </template>
  <n-image src="/share/element-form.png" />
</Popover>
<span>可以发现，只有在子<code>formItem</code>组件的<code>mounted</code>和<code>beforeDestroy</code>时，才会触发<code>form</code>组件的<code>add</code>和<code>remove</code>方法，修改<code>fields</code>数组的内容。</span>
</div>

而通过前面的`patch`过程我们已经知道，在切换的过程中并没有发生`formItem`组件的`mounted`和`beforeDestroy`，而是复用了同一个组件实例，所以`fields`数组中的实例其实一直都是存在的，没有发生变化。

所以既然`fields`数组保存的实例没有变化，那么我们就需要去查看`formItem`本身的校验方法是否出了问题。

</v-clicks>

---

还是查看这张流程图，我们重点关注`formItem`的`validate`方法部分：

<n-image src="/share/element-form.png" class="h-250px" />

可以看到`formItem`在每次校验时，都需要去获取校验的`rules`，也就是规则，规则的定义有多种写法，例如

1. 定义在`form`组件的`rules`对象中，通过`prop`属性区分是哪个`formItem`的规则
2. 直接定义在`formItem`组件的`rules`属性上
3. 定义在`formItem`的`required`字段上

---

查看`formItem`的`validate`方法源码可以发现

```js {3-8}{maxHeight:'400px'}
validate(trigger, callback = noop) {
  this.validateDisabled = false;
  const rules = this.getFilteredRule(trigger);
  // 如果没有获取到规则，则直接校验通过
  if ((!rules || rules.length === 0) && this.required === undefined) {
    callback();
    return true;
  }

  this.validateState = 'validating';

  const descriptor = {};
  if (rules && rules.length > 0) {
    rules.forEach(rule => {
      delete rule.trigger;
    });
  }
  descriptor[this.prop] = rules;

  const validator = new AsyncValidator(descriptor);
  const model = {};

  model[this.prop] = this.fieldValue;

  validator.validate(model, { firstFields: true }, (errors, invalidFields) => {
    this.validateState = !errors ? 'success' : 'error';
    this.validateMessage = errors ? errors[0].message : '';

    callback(this.validateMessage, invalidFields);
    this.elForm && this.elForm.$emit('validate', this.prop, !errors, this.validateMessage || null);
  });
},
```

<v-click>

我们需要去查看页面切换选项后，`form`组件中`fields`保存的`formItem`的`rules`是否可以正常获取到

</v-click>

---

查看结果后可以发现，获取`rules`返回的结果为空，为什么会为空呢

<v-clicks>

上一个问题中我们已经知道，在`patch`过程中，新旧vnode列表存在了错误的复用问题，导致前后DOM元素的位置发生错误。那么其实除了元素的复用导致的问题，其中还存在着组件实例复用引发的问题。

这里给出一个简单的组件vnode`patch`流程图

<n-image src="/share/componentvnodepatch.png" />

在之前的的`patchVnode`示意图中我们已经知道，复用的过程不止存在于普通的元素节点，组件节点也是需要复用的。`patchVnode`过程中，首先会把旧的vnode组件实例直接赋值给新的vnode组件节点，之后把新的vnode组件节点上的属性值（包括`props`，`listeners`，`attrs`等等）赋值给旧的vnode实例，重新去走一遍组件实例的“初始化”流程（处理新赋值的这些属性）。


所以这意味着`form`中`fields`字段保存的`formItem`实例在`patch`后，传入的`prop`和`rules`字段的值已经更新为新的vnode上对应的值

</v-clicks>

---

那么新的vnode上对应的值是什么呢

<v-clicks>

我们应该还记得，这个案例中发生了错误的节点`patch`，导致原来正常可以校验的`formItem`的vnode与原来不需要校验的`formItem`的vnode进行了`patch`

<n-image src="share/patch前后表单.png" class="h-250px" />

那么`prop`和`rules`的字段值也相应的变为新的节点上的值，也就是空。所以在调用校验方法时，并不是不校验，而是没有对应的校验规则去进行校验，自然也就直接返回了校验通过，导致的后续接口调用报错的问题。当然，这个问题已经解决掉了，因为我们解决了第一个错误`patch`的问题，修改后的表单已经可以正常校验了

</v-clicks>
