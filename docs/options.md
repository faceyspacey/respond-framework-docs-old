# Options

Below are the options passed to `createModule` which are regularly customized. 

In the rare case that Respond doesn't do what you want to do it, please see the [advanced-options](./advanced-options.md).

## `location: string | selector`

By default Respond `location` state is stored at `state.location`.  You can customize where it lives **by a string:**

```js
const options = {
  location: 'foo-reducer'
}
```

**or a function:**

```js
const options = {
  location: state => state['somewhere-else']
}
```



## `title: string | selector`


Respond automatically changes the page title to whatever your `title` reducer returns on every route change. You can customize what reducer powers it just like the `location` state:


```js
const options = {
  title: 'title-reducer'
}
```

**or:**

```js
const options = {
  title: state => state['somewhere-else']
}
```

However, you must be using the `changePageTitle` middleware, which comes bundled part of the default middleware. If you customize, here's how to add it to a minimal `middlewares` array:


```js
import { changePageTitle } from 'respond-framework/middleware'

const middlewares [
  transformAction,
  enter,
  changePageTitle, // shocking
  call('thunk', { cache: true }),
]

export default createModule(config, options, middlewares)
```



## `inject(respondApi): Object`

`inject` is Respond's dependency injection mechanism. It offers an opportunity to inject whatever tools you want passed to your callbacks or middleware. The most common example is an `api` object like so:

```js
import Api from '../lib/api'

const config = {
  routes: {
    profile: {
      path: '/profile/:slug',
      thunk: ({ api, info, params }) => api.get(`users/${params.slug}`)
    }
  }
}

export default (request, response) => createModule(config, {
  inject: () => {
    const api = new Api(request, response)
    const info = { foo: 'bar' }
    return { api, info }
  }
})
```

This is the pattern for providing a fully populated `api` instance on both the client and server as part of SSR. See [dependency injection](./dependency-injection.md) for a complete example of how you might provide a feature-complete `api` instance.



## `shouldCall(callbackName, route, request): boolean`

This is a very common function to override. It's used to make the determination of what callbacks to call. Here's the default implementation:

```js
const options = {
  shouldCall: (name, route, req) => {
    if (!route[name] && !req.options[name]) return false

    // skip callbacks (beforeEnter, thunk, etc) called on server, which produced initialState
    if (isHydrate(req) && !/onEnter|onError/.test(name)) return false

    // dont allow these client-centric callbacks on the server
    if (isServer() && /onEnter|Leave/.test(name)) return false

    return allowBoth
  }
}

const allowBoth = { route: true, options: true }
```

Return `false` to allow none, or an object as above to specify which of the 2 to call. 

A common pattern is to turn off parallel calling of both route callbacks + options callbacks, and use the options callback as a "fallback." In other words to give precedence to a `route` callback. Here's how you might implement that:

```diff
options.shouldCall = (name, route, req) => {
  if (!route[name] && !req.options[name]) return false

  // skip callbacks (beforeEnter, thunk, etc) called on server, which produced initialState
  if (isHydrate(req) && !/onEnter|onError/.test(name)) return false

  // dont allow these client-centric callbacks on the server
  if (isServer() && /onEnter|Leave/.test(name)) return false

+  if (route[name]) return allowOnlyRoute
  return allowBoth
}

const allowBoth = { route: true, options: true }
+const allowOnlyRoute = { route: true, options: false }
```

And here's how to let the route have a simple boolean flag to skip options callbacks:

```js
return { options: !route.skipOpts, route: true }
```



## `basenames: string[]`

Respond both auto-detects basenames and lets you change the basename like any other URL feature. However, they must be known up front. Eg:

```js
options.basenames = ['en', 'fr', 'jp', 'ru', 'vi', 'sp']
```

If any URL starts with one of those, the corresponding basename will automatically be set in actions and state. For example, if a user directly visits: `https://www.yourapp.com/jp/login`

Then in state and actions you will see:

- `state.location.basename: 'jp'`

- `action.basename: 'jp'`,

`/jp` will continue to be prepended to the URL for *all subsequent* route transitions, and without you having to specify it in your actions. If you want to change it you can do:

```js
const LanguageSelector = (props, state, actions ) => 
  <Dropdown options={state.location.basenames} onChange={bn => actions.changeBasename(bn)} />
```

You could also change the basename while dispatching another action:

```js
const MyComponent = (props, state, actions ) => 
  <Button onClick={actions.login({ basename: 'sp' })} />
```

Then `/sp` will be prepended to all URLs going forward, *and without you having to continue to specify the basename.* It's seamless and automatitc. **Basenames aren't an afterthought requiring workarounds with Respond.**

That means, they don't affect the action type. The same route/type can be visited via different basenames. A route's `path` is only considered after any known basenames have been prefixed to paths (or none at all).  You can set the basename to an empty string to remove it.



## `parseSearch(string): object`

Provide this option to customize how search strings are parsed. Here's how the default `parseSearch` is implemented:

```js
import qs from 'qs'

const decoder = (str, decode) => isNumber(str) ? parseFloat(str) : decode(str)
const isNumber = (str) => !isNaN(str) && !isNaN(parseFloat(str))

export default (search) => qs.parse(search, { decoder })
```


## `stringifyQuery(Object): string`

Provide this option to customize search strings are generated. Here's how the default `stringifyQuery` is implemented:

```js
import qs from 'qs'
export default (obj) => qs.stringify(obj)
```


## `initialEntries: Array<Entry>`


`initialEntries` are used during SSR and in React Native.

**Example:**

```js
export default (request, response) => createModule(config, {
  initialEntries: [request.url]
})
```

An Entry has these 3 different forms:

- path string *(`'/login'`)*
- array containing a path, state object, and optionally a unique key to identify it *(`['/login', { foo: 'bar' }, 123456]`)*
- an action object (`{ type: 'LOGIN' }`)

During SSR and when handling deep links on React Native, it's typically and simplest to just pass the detected URL. 

> One pattern in React Native is to make push notifications also confirm to the same interface by transforming the info contained within a notification to a either an action object or a URL.


**During SSR, you almost never want to provide more than one entry.** Instead fill in the appropriate default entries at the route level, eg: `route.initialEntries`. See [route-config#initialEntries](./route-config#initialEntries) for a deep dive on this topic.

Client side, the initial URL is always automatically detected using standard browser APIs. You never need to provide this option client side, even in a full stack universal app.


## `initialIndex: number`

If you provide multiple entries in `initialEntries`, you can actually set what index the user is at on the history track, eg:


```js
export default (request, response) => createModule(config, {
  initialEntries: ['/', '/login', request.url, '/dashboard',],
  initialIndex: 2
})
```

This is especially useful when you're using Stack Navigators, as on React Native. Again, the same option also exists at the route level at: [route-config#initialIndex](./route-config#initialIndex).





## `initialN: number`

Set the direction the user was going along the history track to positive one (default) or negative one:


```js
export default (request, response) => createModule(config, {
  initialN: -1
})
```

This information is used to populate state for the previous route at `state.location.prev`. It also exists at the route level.



## `save(locationState): void`

This is used primarily on React Native to save the entire location state for restoration on subsequent openings of your app.

```js
options.save = ({ entries, index, n }) => {
  entries = entries.map(e => [e.location.url, e.state, e.location.key]) // condense data
  saveSomehow({ entries, index, n })
}
```


## `restore(api, entries?, index?, n?): Array<Entries>`

This is used primarily on React Native to restore location state that was saved the last time the app was open. It's passed the `initialEntries/index/n` you provide. Use it to transform those entries in a custom way, overriding the default.

```js
import { toEntries } from 'respond-framework/utils'

options.restore = (api, entries, index, n) => {
  // you can perform a custom transformation here
  return toEntries(api, entries, index, n)
}
```

> You can also completely disregard the passed `entries/index/n`, and retreive your own that you possibly saved somewhere else via `saveSomehow` above.

Client side on the web, you will only ever receive an `api` object, and it's your job to retreive the entries from somewhere else, usually `localStorage`. This is a highly advanced task. Hopefully you will never need this. If you absolutely do, you will want to checkout our `BrowserHistory` class.


