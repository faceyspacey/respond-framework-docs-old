# Dependency Injection

Standard procedure for injecting an `api` instance (or any other such utility) into callbacks goes something like this:

```js
import Api from './Api'

export default (request, response) => createApp({
  routes,
  components,
  reducers,
  etc,
  options: {
    inject: {
      api: new Api(request, response),
      miscContext: { foo: 'bar' },
      etc
    }
  }
})
```

> you will want to make `request` and `response` available on server; they are needed to access to the Express APIs for cookie handling. `request` in this case is obviously an Express `request`, not a Respond one.

Now, in your route callbacks, the key/vals of `inject` will be merged into the **Respond `request`** passed to callbacks and available like so:

```js
SUBMIT_LOGIN: {
  thunk: async (request, action) => {
    const { api } = request
    const { email, password } = action.payload
    const { success, message } = await api.login(email, password) // token received

    return success ? actions.dashboard() : actions.flash(message)
  }
}
```


## Api Class

This is how you might implement the `Api` class. The important part is how your access token is made transparently/automatically available to all requests once the user logs in:


```js
import axios from 'axios' // popular data fetching library
import Cookies from './cookies'
import config from '../config'

axios.defaults.baseURL = `${config.rootUrl}/`

export default class Api {
  constructor(req, res) {
    this._cookies = req
      ? new Cookies(req.headers.cookie, this.createHooks(req, res)) // server
      : new Cookies() // client (cookies discovered client-side by this class)
  }

  createHooks(req, res) {
    return {
      onSet(name, value, opts) {
        if (!res.cookie || res.headersSent) return

        const options = {
          path: '/',
          expires: new Date(2080, 1, 1, 0, 0, 1),
          maxAge: opts.maxAge || 2147483647,
          ...opts
        }

        res.cookie(name, value, options)
      },
      onRemove(name, opts) {
        if (!res.clearCookie || res.headersSent) return

        const options = {
          path: '/',
          maxAge: 0,
          ...opts
        }

        res.clearCookie(name, options)
      }
    }
  }

  setCookie(name, value) {
    this._cookies.set(name, value)
  }

  getCookie(name) {
    return this._cookies.get(name)
  }

  removeCookie(name) {
    this._cookies.remove(name)
  }

  config() {
    return (
      this.getCookie('token') && {
        headers: {
          Authorization: `Bearer ${this.getCookie('token')}`
        }
      }
    )
  }

  get(url, params, conf) {
    const config = { ...this.config(), params, ...conf }
    return axios.get(url, config)
  }

  post(url, data, conf) {
    const config = { ...this.config(), ...conf }
    return axios.post(url, data, config)
  }

  async login(email, password) {
    const { token, message } = await axios.post('login', { email, password })

    if (token) {
      this.setCookie('token', token)
      return { success: true }
    }

    return { message }
  }
}
```


## Cookie Class

And for the `Cookie` class you might have something like this using the [universal-cookies](https://www.npmjs.com/package/universal-cookie) package on npm:

```js
import UniversalCookie from 'universal-cookie'

export default class Cookie extends UniversalCookie {
  set(name, value, opts) {
    const options = {
      ...opts,
      path: '/',
      expires: new Date(2080, 1, 1, 0, 0, 1),
      maxAge: 2147483647
    }

    return super.set(name, value, options)
  }

  remove(name, opts) {
    const options = {
      ...opts,
      path: '/',
      maxAge: 0
    }

    return super.remove(name, options)
  }
}
```

The reason you use cookies is because `localStorage` isn't available server-side. Therefore, cookies are the only *universal* solution, as they're available on both the client and the server. 

Client-only single page apps could just as well use `localStorage`, of which there are many similar examples on the web you can learn from.