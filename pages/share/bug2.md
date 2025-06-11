## 表单的校验方法失效

第二个问题就相对简单了，因为问题范围已经确定，就是发生在`form`组件中，我们只要从`form`组件的`validate`方法入手即可。

表单的校验是通过`element-ui`的`form`组件实现的，调用`form`组件的`validate`方法，触发校验流程，返回校验结果。

查看`validate`的方法源码可以发现

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
