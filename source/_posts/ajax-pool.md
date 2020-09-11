---
title: 合并重复请求
---
在工作中遇到一种情况，前端的一些请求需要携带 token 完成认证，而这个 token 在一段时间内是保持不变的，所以最佳的方式应该是请求一个 token 之后将其存储起来，在未过期的时候可以重复使用。

```javascript
let token
async getToken() {
  if (!token || Date.now() - token.timestamp > 3600 * 1000) {
    const res = await axios.get('/token')
    token = res.data
  }
  return token
}

async function requestWithToken() {
  const token = await getToken()
  return await axios(arguments)
}
```

在一个页面中如果有多个请求时，`getToken`方法会首先查询是否有未过期的 token ，否则就会从后端获取，这样就避免了浪费。

---

然而在页面加载时短时间内多次调用`requestWithToken`方法时，第一次的请求还未返回，token 对象依旧为空，所以之后依旧会发送调用`getToken`方法。

我首先想到的是设置一个 pending 状态表示 token 正在获取中

```javascript
let token
let pending = false
async function getToken () {
  pending = true
  if (!token || Date.now() - token.timestamp > 3600 * 1000 || !pending) {
    const res = await axios.get('/token')
    pending = false
    token = res.data
  }
  return token
}
```

---

但是这样做的话虽然避免了重复发送请，但是获取返回的值却有可能是空的 token ，所以我们还需要额外的工作，需要请求 token 的时候先查询是否存在正在发送的请求，如果是则将方法挂起，等待请求完成，请求完成之后再返回 token

```javascript
let token
async function getToken () {
  if (!token || Date.now() - token.timestamp > 3600 * 1000) {
    token = await requestPool('/token')
  }
  return token
}

const pool = {}
function requestPool(url) {
  if (!pool[url]) {
    pool[url] = []
    return new Promise((resolve, reject) => {
      axios.get(url)
        .then(res => {
          resolve(res)
          pool[url].forEach(cb => cb.resolve(res))
        })
        .cath(e => {
          reject(e)
          pool[url].forEach(cb => cb.reject(e))
        })
        .finally(() => {
          Reflect.delete(pool, url)
        })
    })
  }

  return new Promise((resolve, reject) => {
    queue.push({resolve, reject})
  })
}
```

ps：浏览器本身自带缓存 get 请求的功能，但是受制于后端限制，可能存在个别接口 http headers 无法正确设置的情况，所以还需要前端做些额外的工作来进行缓存。利用这种方法甚至还可以缓存 post 请求。