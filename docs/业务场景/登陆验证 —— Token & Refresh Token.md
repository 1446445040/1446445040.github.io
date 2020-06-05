---
title: 登陆验证 —— token & refresh token
---

# 背景

最近在做管理系统项目的时候，涉及到登陆验证的问题。由于该系统的实时性要求并不高，排除了session + cookie的传统方案，纠结再三，最终选择了使用token进行验证。如此，服务端无需存储会话信息，将信息加密保存在token中，只要密钥没有泄露，安全性还是可以得到保障的。

[Github 仓库](https://github.com/1446445040/competition-system)

技术栈

- 前端：vue全家桶 + ant-design-vue
- 后端：MongoDB + TypeScript + express

欢迎有志之士交流探讨。

# Token

token认证的基本流程是：客户端登陆成功，服务端返回token，客户端存储token。以后的每次请求，都将携带token，一般会放在请求头中，随请求发送：`Authorization: token`。

token的特点是无法废除，即一个token只要颁发了，在有效期内始终是有效的，即使颁发了新的token也不影响原有的token使用。出于安全性考虑，**token的有效时间应该设置的短一些**，通常设置为`30min ~ 1h`。

这样一来每隔这么久就要重置一次token，可以通过refresh token的方式来更新token。

# Refresh Token

refresh token与token一样，都是一段加密字符串，不同的是，refresh token是用来获取新的token的。

在使用成熟的网站、社区时，常常发现很长一段时间我们都不需要重新登陆，这貌似有悖于**token的有效时间应该设置的短一些**。实际上使用refresh token后，登录权限验证的流程变成了下面这样：

登陆成功后服务端返回token和refresh token，客户端存储这两个信息。平时发送请求时携带token，当token过期时服务端会拒绝响应，此时发送一个携带refresh token的新请求，服务端验证后返回新的token，客户端替换旧的token后重新发起请求即可。若refresh token过期，则重新登陆。

可以看出refresh token作用于长期，仅用于获取新的token，同时也控制着用户登录的最长时间，一般会设置的长一些，例如7天，14天，过期之后必须重新登录；token作用于短期，用于获取数据。

# 问题

基本的流程了解清楚后，可以引出以下几个问题：

- token里面放什么？
- 重新请求会不会打断用户的操作？（用户体验差）
- 并发请求如何处理？（会反复调用API刷新，效率低下）

## token里面放什么

token里啥都可以放，只要你愿意。**一般会包含简短的用户信息和过期时间**。。由于token内放的数据是可以解密的，所以**千万不要放敏感信息**如密码等。过期时间用来验证token是否过期，简短的用户信息用于可能的数据库操作（如果有的话）。

## 用户体验

试想如果由于token过期导致用户好不容易填完的表单数据丢失，用户一定会暴跳如雷吧？刷新token一定要考虑用户体验问题。

通常我们会设置全局的拦截（以axios为例）。设置全局响应拦截，通过约定好的信息（如状态码）判断属于哪种情况，最后根据情况采取不同的操作（是刷新token？还是重新登陆？），之后再重新发送之前的请求。

提升用户体验的关键在于，**不能中断当前请求，而是使用新的请求替换原来的请求**。这一点再axios中可以轻松实现，下文会示例。

## 并发请求处理

并发请求时，若恰好token过期，则最终会发起多个刷新token的请求，多余的请求除了增加服务器的压力，没有任何做用。

浏览器中，发出请求时候会开启一条线程。请求完成之后，将对应的回调函数添加到任务队列中，等待 JS 引擎处理。而我们需要整合这个过程，将并发请求拦截汇总，最终只发出一次刷新请求。这便涉及线程同步的问题，我们可以通过**加锁，和缓冲**来解决。

- 加锁：简单来说就是在模块内部设置一个全局变量，用来标志当前的状态，防止冲突。

- 缓冲：就是设置一个空间，将当时发生但来不及处理的内容存储起来，在合适的时机再处理。

干说不好理解，下面上代码。

# 实现

**约定：403重新登陆，401需要刷新**

## 前端

使用`vue + axios`实现，首先封装一下全局的`axios API`

```javascript
/**
 * axios.js
 */
import { message } from 'ant-design-vue'
import axios from 'axios'
import store from '../store'
import router from '../router'
import handle401 from './handle401'

axios.defaults.baseURL = '/api'

// 请求携带token
axios.interceptors.request.use(config => {
  // 判断是为了防止自定义header被覆盖
  if (!config.headers.authorization && store.state.token) {
    config.headers.authorization = store.state.token
  }
  return config
})

// 若因401而拒绝，则刷新token，若403则跳转登录
// 返回的内容将会替换当前请求（Promise链式调用）
axios.interceptors.response.use(null, error => {
  const { status, config } = error.response
  if (status === 401) {
    return handle401(config)
  } else if (status === 403) {
    message.warn('身份凭证过期，请重新登录')
    router.replace({ name: 'login' }).catch(e => e)
  }
  return Promise.reject(error)
})

export default axios // 导出axios对象，所有请求都使用这个对象

```

然后是对于401状态的处理，细心的小伙伴可以发现，这里存在`handle404.js`和`axios.js`的循环引用问题，感兴趣的可以戳 [阮一峰 —— ES6模块加载](https://es6.ruanyifeng.com/#docs/module-loader#%E5%BE%AA%E7%8E%AF%E5%8A%A0%E8%BD%BD)，由于不会影响代码逻辑的正常执行，这里不做展开。

```javascript
/**
 * handle404.js
 */
import store from '../store'
import axios from './axios'
import { REFRESH_TOKEN } from '../store/mutation-types'

let lock = false // 锁
const originRequest = [] // 缓冲

/**
 * 处理401——刷新token并处理之前的请求，目的在于实现用户无感知刷新
 * @param config 之前的请求的配置
 * @returns {Promise<unknown>}
 */
export default function (config) {
  if (!lock) {
    lock = true
    store.dispatch(REFRESH_TOKEN).then(newToken => {
      // 使用新的token替换旧的token，并构造新的请求
      const requests = originRequest.map(callback => callback(newToken))
      // 重新发送请求
      return axios.all(requests)
    }).finally(() => {
      // 重置
      lock = false
      originRequest.splice(0)
    })
  }
  // 关键代码，返回Promise替换当前的请求
  return new Promise(resolve => {
    // 收集旧的请求，以便刷新后构造新的请求，同时由于Promise链式调用的效果，
    // axios(config)的结果就是最终的请求结果
    originRequest.push(newToken => {
      config.headers.authorization = newToken
      resolve(axios(config))
    })
  })
}
```

这是接口：

```javascript
/**
 * index.js
 */
import axios from './axios'

export const login = data => axios.post('/auth/login', data)
export const refreshToken = originToken => {
  return axios.get('/auth/refresh', {
    headers: {
      authorization: originToken
    }
  })
}
```

然后是`vuex`的相关代码：

```javascript
import { message } from 'ant-design-vue'
import { LOGIN, REFRESH_TOKEN } from './mutation-types'
import { login, refreshToken } from '../api/index.js'

export default {
  [LOGIN] ({ commit, state }, info) {
    ···
  },
  [REFRESH_TOKEN] ({ commit, state }) {
    // 使用Promise包装便于控制流程
    return new Promise((resolve, reject) => {
      refreshToken(state.refreshToken).then(({ data: newToken }) => {
        commit(REFRESH_TOKEN, newToken)
        resolve(newToken)
      }).catch(reject)
    })
  }
}

```

##  后端

使用` express + jsonwebtoken`实现

为了便于演示，token过期时间设置为10s，refresh token过期时间设置为20s。

``` typescript
/**
 * token.ts
 */
import dayjs from 'dayjs'
import { sign } from 'jsonwebtoken'
import secretKey from '../config/tokenKey'

// 控制普通token，客户端过期后无需再次登录
export const getToken = function () {
  return sign({
    exp: dayjs().add(10, 's').valueOf()
  }, secretKey)
}

// 控制客户端最长登陆时间，超时重新登录
export const getRefreshToken = function (payload: any) {
  return sign({
    user: payload, // 这里放入一点用户信息，刷新的时候用来查数据库，简单的验证一下。
    exp: dayjs().add(20, 's').valueOf()
  }, secretKey)
}

```

登录路由部分代码：

``` typescript
/**
 * login.ts
 */
import { Router } from 'express'
import { getRefreshToken, getToken } from '../../utils/token'

const router = Router().
router.post('/auth/login', function (req, res) {
    ...
    res.json({
        code: 0,
        msg: '登陆成功',
        data: {
            user: { identity, ...user },
            token: getToken(),
            refreshToken: getRefreshToken({ identity, account })
        }
    })
    ...
})

export default router
```

`resfresh token`路由

```typescript
/**
 * refresh.ts
 */
import { Router } from 'express'
import dayjs from 'dayjs'
import { verify } from 'jsonwebtoken'
import { find } from '../../db/dao'
import { USER } from '../../db/model'
import secretKey from '../../config/tokenKey'
import { getToken } from '../../utils/token'

const router = Router()

router.get('/auth/refresh', function (req, res) {
  const refreshToken = req.headers.authorization
  if (!refreshToken) {
    return res.status(403).end()
  }
  verify(refreshToken, secretKey, function (err, payload: any) {
    // token 解析失败，重新登录
    if (err) {
      return res.status(403).end()
    }
    const { exp, user } = payload
    // refreshToken过期，重新登录
    if (dayjs().isAfter(exp)) {
      return res.status(403).end()
    }
    // 否则刷新token
    find(USER, user).then(users => {
      if (users.length === 0) {
        res.status(403).end()
      } else {
        res.status(200).send(getToken())
      }
    }).catch(e => {
      res.status(500).end(e.message)
    })
  })
})

export default router
```

登陆验证中间件：

```typescript
/**
 * loginChecker.ts
 */
import dayjs from 'dayjs'
import { Request, Response, NextFunction } from 'express'
import { verify } from 'jsonwebtoken'
import secretKey from '../config/tokenKey'

export default function (req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization
  if (!token) {
    return res.status(403).end()
  }
  verify(token, secretKey, function (err, payload: any) {
    if (err) {
      return res.status(403).end(err.message)
    }
    const { exp } = payload
    console.log(dayjs(exp).format('YYYY-MM-DD HH:mm:ss'))
    if (dayjs().isAfter(exp)) {
      res.status(401).end('Unauthorized') // 过期，401提示客户端刷新token
    } else {
      next() // 否则通过验证
    }
  })
}
```

# 总结

这个登陆验证的流程中，最值得细细品味的，当属**并发请求处理**的部分，即`handle404.js`文件中的内容。它涉及并发问题，而Vue中也有类似的问题， 如视图更新：

vue中数据变化触发的视图更新是异步的，这使得短时间内数据的多次变化可以整合到一起，避免渲染无意义的中间态。其内部也是使用一个标志量和一个缓冲区来实现的。

文章如有纰漏，欢迎批评指正。

# 参考

[请求时token过期自动刷新token](https://segmentfault.com/a/1190000016946316)