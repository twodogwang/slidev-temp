## 表单的校验方法失效

第二个问题就相对简单了，因为问题范围已经确定，就是发生在`form`组件中，我们只要从`form`组件的源码入手即可。`element-ui`的表单校验大家都很熟悉了，主要流程是利用`form-item`组件先在template中定义好需要校验的数据字段值还有校验的方法`rules`，最后通过执行`form`组件的`validate`方法，触发校验流程，返回校验结果。所以我们从校验方法的触发开始入手，先来查看`validate`方法的源码

查看`validate`的方法源码可以发现，内部其实非常简单，除了必要的参数的值的校验和提示信息，主要的校验流程只有调用保存的`fields`数组中各自的`validate`方法而已，所以我们排查的范围缩小到了查看`fields`中保存的内容是什么

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

---

这里给出一张`form`组件和`form-item`组件在校验实现上的简略流程图。

<n-image src="/share/element-form.png" />

前面提到的重点，`form`组件的`fields`数组其实保存的就是`form-item`组件，每个`form-item`在`mounted`之后，会把自身保存在`form`的`fields`中，在`form`校验时，其实就是触发每个表单项的校验方法。
