## Promise 

## Promise 规范
```
  const promise1 = new Promise((resolve, reject) => {
    reject()
  })

  promise1
  .catch(() => {
    console.log('catch')
    throw 'catch throw error'
  })
  .then(null, function () {
    console.log('reject')
    return 123
    // throw 'err'
  })
  .then(null, null)
  .then(null, null)
  .then((res) => {
    console.log('promise 已完成', res)
  }, () => {
    console.log('promise 已拒绝')
  })
```

## Promise 的简易实现
