## 表单的校验方法失效

第二个问题就相对简单了，因为问题范围已经确定，就是发生在`form`组件中，我们只要从`form`组件的源码入手即可。`element-ui`的表单校验大家都很熟悉了，主要流程是利用`form-item`组件先在template中定义好需要校验的数据字段值还有校验的方法`rules`，最后通过执行`form`组件的`validate`方法，触发校验流程，返回校验结果。所以我们从校验方法的触发开始入手，先来查看`validate`方法的源码

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

查看`validate`的方法源码可以发现，内部其实非常简单，除了必要的参数的值的校验和提示信息，主要的校验流程只有调用保存的`fields`数组中各自的`validate`方法而已，所以我们排查的范围缩小到了查看`fields`中保存的内容是什么

</v-click>

---

这里给出一张`form`组件和`form-item`组件在校验实现上的简略流程图。

<n-image src="/share/element-form.png" class="h-350px" />

前面提到的重点，`form`组件的`fields`数组其实保存的就是`form-item`组件，每个`form-item`在`mounted`之后，会把自身保存在`form`的`fields`中，在`form`校验时，其实就是触发每个`form-item`的校验方法。

---

那么我们自然会考虑到，是否是在切换了选项后，原来的`form`组件的`fields`数组发生了变化，原来可用的`form-item`组件被移除了，导致校验方法失效了呢？

<v-clicks>

<div>
<span>仔细查看<code>form</code>的<code>fields</code>数组中数据的</span>
<Popover width="600">
  <template #trigger>
    <span>变化时机</span>
  </template>
  <n-image src="/share/element-form.png" />
</Popover>
<span>可以发现，只有在子<code>form-item</code>组件的<code>mounted</code>和<code>unmounted</code>时，才会触发<code>form</code>组件的<code>add</code>和<code>remove</code>方法，修改<code>fields</code>数组的内容。</span>
</div>

而通过前面的`patch`过程我们已经知道，在切换的过程中并没有发生`form-item`组件的`mounted`和`unmounted`，而是复用了同一个组件实例，所以`fields`数组中的实例其实一直都是存在的，没有发生变化。

所以既然`fields`数组保存的实例没有变化，那么我们就需要去查看`form-item`本身的校验方法是否出了问题。

</v-clicks>

---

还是查看这张流程图，我们重点关注`form-item`的`validate`方法部分：

<n-image src="/share/element-form.png" class="h-350px" />

可以看到`form-item`在每次校验时，都需要去获取校验的`rules`，也就是规则。规则的定义有多种写法，例如

1. 定义在`form`组件的`rules`对象中，通过`key`区分是哪个`form-item`的规则
2. 直接定义在`form-item`组件的`rules`属性上
3. 定义在`form-item`的`required`字段上
