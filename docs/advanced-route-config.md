# Advanced Route Config

Respond offers some additional capabilities on your `route` that you don't need to think about when starting out, but will inevitably save you time down the line. Let's check these configuration options out:



## `route.redirect`

Respond provides a shortcut for redirecting that doesn't require using a callback. Just specify the name of a different route to redirect to:

```js
routes: {
  OLD: {
    path: '/foo/:param',
    redirect: 'BAR'
  },
  NEW: {
    path: '/bar/:param',
  }
}
```

Make sure to name the parameters the same. For example this is wrong:

```js
path: '/foo/:name',
```

whereas this is correct:

```js
path: '/foo/:param',
```

because the `NEW` route is expecting a parameter by that name.

You can also specify the types of namespaced child modules:

```js
OLD: {
  path: '/foo/:param',
  redirect: 'CHILD_MODULE/BAR'
}
```


## `route.inherit`

If you ever find yourself re-using the same callbacks across similar routes, you can specify the name of another route to inherit all its callbacks, eg:

```js
routes: {
  FOO: {
    path: '/foo',
    beforeEnter: () => ...,
    thunk: () => ...,
    onEnter: () => ...,
  },
  BAR: {
    path: '/bar',
    inherit: 'FOO'
  }
}
```

You can also override any of them:

```js
BAR: {
  path: '/bar',
  inherit: 'FOO',
  onEnter: () => ...,
},
```

Like `route.redirect` you can specify namedspaced children types:

```js
BAR: {
  path: '/foo/:param',
  inherit: 'CHILD_MODULE/FOO'
},
```


## `route.callback: string`

Similar to `route.inherit` but for a single callback:

```js
routes: {
  FOO: {
    path: '/foo',
    thunk: () => ...,
    onEnter: () => ..., // won't be inherited
  },
  BAR: {
    path: '/bar',
    thunk: 'FOO'
  }
}
```


## `route.callback[]`

If you have tasks you're repeating across different callbacks, you can attain some re-use through the array callback form:


```js
import { track, doSomethingElse } from './callbacks'

routes: {
  FOO: {
    path: '/foo/:param',
    thunk: [
      track,
      doSomethingElse,
      (req) => {
        req.passOn = true
      },
      ({ api, passOn }, { params }) => passOn && api.get(`foo/${param}`)
    ]
  }
}
```

If you want them to run in parallel, there's an option for that:

```js
routes: {
  FOO: {
    path: '/foo/:param',
    thunk: [
      track,
      doSomethingElse,
      ({ api }, { params }) => api.get(`foo/${param}`)
    ],
    parallel: true
  }
}
```

You can also combine inherit strings and the array form:

```js
routes: {
  FOO: {
    path: '/foo/:param',
    thunk: [
      'BAR',
      'BAZ',
      ({ api }, { params }) => api.get(`foo/${param}`)
    ],
    parallel: true
  },
  BAR: {
    path: '/bar',
    thunk: () => ...,
  },
  BAR: {
    path: '/bar',
    thunk: () => ...,
  }
}
```

> Yes, you can inherit another array of callbacks as well.

## `route.middleware`

Respond also lets you override the default middleware that runs for each route. Typically instead of using callbacks on the route, you do all your work in the middleware itself:

```js
import analytics from './analytics'
import { commit, changePageTitle } from 'respond-framework/middleware'

routes: {
  FOO: {
    path: '/foo',
    middleware: [
      api => async (req, next) => {
        const { api, actions } = req
        const payload = await api.fetch('something')
        actions.foo.complete(payload)

        await next()
        analytics.track('route', req)
      },
      commit,
      changePageTitle
    ]
  }
}
```

You have to call and await `next` for the following middleware to be called (in this case `commit` and `changePageTitle`). This allows for re-use of existing middleware. 

If you don't plan to do anything after the subsequent middleware are called you can simply return the call to `next` since it returns a promise that the middleware pipeline can resolve on its own:

```js
api => async (req, next) => {
  const { api, actions } = req
  const payload = await api.fetch('something')
  actions.foo.complete(payload)

  return next()
},
```

If you're middleware doesn't do anything asynchronous, you can remove the `async` tag:

```js
api => (req, next) => {
  analytics.track('route', req)
  return next()
},
```

If you have setup work you only do once on load of the app, you can do it like so:

```js
api => {
  const entryUrl = api.history.url            // only run once on app startup

  return (req, next) => {                     // called every time this route is matched
    analytics.track('route', req, entryUrl)
    return next()
  }
}
```

> In addition to middleware customization, `route.middleware` can be a good way to reduce the number of things that go on during route transitions. This provides you additional control over *(and certainty of)* everything going on. It's basically an advanced *all-in-one* callback.

## `route.initialEntries: Array<Entry>`

> `initialEntries` at the route level is only kicked into action if the given route is the first route the app is loaded on, ***and there isn't more than one entry loaded either from the options level or from the browser's `sessionStorage`.*** In other words, it's used for direct visits to your app or site.


The token example where this capability is needed at the route level is if a user directly visits, say, step 3 of your checkout. `route.initialEntries` allow you to fill in steps 1-3, so if the user taps *back* the correct entries will be there. 

It will be your app's responsibility to fill it in with default information, in case the user doesn't go back, but just proceeds to complete the checkout. Strategies to fill out that info include using [redux-persist](https://github.com/rt2zz/redux-persist) or on app startup simply loading user info asynchronously from the server and then storing it in the store.

In web clients, Respond replicates the desired stack of `initialEntries` in the address bar, insuring the browser's hidden stack of entries matches how your app perceives it. 

> If you're new to web development, you'll recognize history mirroring and tracking as a *world first* feature that's usually considered impossible. **Respond is known for making the impossible *possible*. Happy coding, *first Responders*.**


See [options.initialEntries](./options.md#initialentries-arrayentry) for more information. Unlike most other route level configurations, the options level one overrides the route level one as described above.


## `route.initialIndex: number`

See [options.initialIndex](./options.md#initialindex-number) for more information. The options level one overrides the route level.


## `route.initialN: number`

See [options.initialN](./options.md#initialn-number) for more information. The options level one overrides the route level.


## `route.parseSearch`

See [options.parseSearch](./options.md#parsesearchstring-object). The route level version overrides the option level one.


## `route.stringifyQuery`

See [options.stringifyQuery](./options.md#stringifyqueryobject-string). The route level version overrides the option level one.