# Middleware

The crown jewel of Respond framework is its middleware pipeline. There's endless things you can do with it. It greatly simplifies your app, providing some much needed best practices and architecture. 

The pipeline linearly consolidates related work that otherwise ends up confusingly sprinkled throughout your component tree. 

It will provide you great developer happiness if any of the following are true:

- You're tired of pulling your hair out wondering where xyz is happening in your component tree.
- You're tired of surprises surfacing as components render, and desire more predictability.
- You remember the server-side MVC days, and you seek a return to normalcy.
- You recognize the stability provided by sequential ordering of effects as in traditional software, and you recognize it's about time the pendulum swings back into balance for Reactlandia.
- **You understand that data deps *can be known* by the top level URL. *Therefore it makes no logical sense to complicate things by splitting up such work across a component tree of unknown (and very likely deep) depth*.**


Regarding the last point, there are near zero apps that can't determine their data dependencies based on a URL. The only requirement is that you URL-ize every state, which Respond makes frictionless. In other words, provide a *"uniform resource indicator"* for each state, which makes a lot of sense to do anyway, and comes with many additional benefits. 

> If you have an example of an app that can't hoist data dependencies per route, please describe it in [this public spreadsheet](https://docs.google.com/spreadsheets/d/1uf4XH83Jjur8iGfGqtlLxf2pxetQvOJxHpUkJToc2D0/edit?usp=sharing) which we're using to collate such examples.

> If you're wondering about the enormous number of states your app can be in, keep in mind that with Respond your app will have far fewer states, and instead appropriately *respond* to the 1-2 actions dispatched per route transition.

Customizing your middleware is how you bring conventions/architecture to your app, while simplifying it. Let's take a look:



## Default Middleware 

Respond comes with the following default middleware pipeline:

```js
import { call, transformAction, commit, etc } from 'respond-framework/middleware'

middlewares: [
  serverRedirect,      
  anonymousThunk,
  pathlessRoute('thunk'),
  transformAction,
  call('beforeLeave', { prev: true }),
  call('beforeEnter'),
  commit,
  changePageTitle(state => state.title),
  call('onLeave', { prev: true }),
  call('onEnter'),
  call('thunk', { cache: true }),
  call('onComplete')
]
```

You can customize the middleware used via the 3rd argument to createApp:

```js
const middlewares = [
  call('beforeEnter'),
  commit,
  call('thunk', { cache: true })
]

createApp(config, options, middlewares)
```


`call` is a provided middleware you can use to specify your own conventions. With these usages of the `call`, you can expect to have available these matching callbacks on your routes:

```js
routes: {
  PRIVATE: {
    path: '/private',
    beforeEnter: request => ...,
    thunk: request => ...,
  }
}
```


Individual modules can also have unique sets of middleware:
```js
createModule(config, options, middlewares)
```


You can even override the middleware for each route:

```js
routes: {
  FOO: {
    path: '/foo',
    middleware: [...]
  }
}
```



## Built-in Middleware

Even when you provide your own middleware array, there are a few middlewares that are built-in and are prepended to yours:

```js
middlewares: [
  serverRedirect,      
  anonymousThunk,
  pathlessRoute('thunk'),
  transformAction,
  // ...whatever you specify
```

Therefore, if you specified the custom middleware array from the last section, you actually get this:

```diff
middlewares: [
+  serverRedirect,      
+  anonymousThunk,
+  pathlessRoute('thunk'),
+  transformAction,
  call('beforeEnter'),
  commit,
  call('thunk' { cache: true })
]
```

That's not something you need to concern yourself with. It's just an example of Respond keeping a **"slim core"** while re-using its own plugin API (i.e. a hallmark of good software design).

> If you find yourself in the rare position where you need to remove or customize some of the built-in middleware, [options.compose](./advanced-options.md#compose-middlewares-api-killonredirect-false-request-promiseany) offers an escape hatch.



## Customizing Middleware

Understanding how the middleware pipeline works is no big deal. Let's customize the middleware at the route level to gain an understanding of a minimal middleware pipeline:

```js
import { commit } from 'respond-framework/middleware'

FOO: {
  path: '/foo',
  middleware: [
    api => async (request, next) => {
      const { api, actions } = request
      const payload = await api.fetch('something')
      actions.foo.complete(payload)

      await next() // trigger the next middleware ("commit")
      // perhaps do something after subsequent middleware runs
    },
    commit
  ]
}
```

As you can see, you only need to insure `commit` is called at some point to insure store state changes and the browser history updates. It's not too different to providing a `thunk` callback--let's look at that to compare:


```js
FOO: {
  path: '/foo',
  thunk: ({ api }) => api.fetch('something') // actions.foo.complete automatically called
}
```

Here you're not responsible for calling the *next* middleware in the chain, and the callback signature is simplified. In addition, returns are automatically dispatched as the payload to an action with the type `FOO.complete`.

So what are the reasons you might want to customize the middleware pipeline for an individual route?

1. you want deeper/full control over everything that happens in a route transition
2. you want to use a different set of middleware than used for the rest of your routes

The second point is self-explanitory, so let's take a look at the first point:

```js
import analytics from './analytics'

FOO: {
  path: '/foo',
  middleware: [
    api => async (request) => {
      const { api, actions, commit } = request
      const payload = await api.fetch('something')
      actions.foo.complete(payload)

      await commit()
      analytics.track('transition', request)
    }
  ]
}
```

Notice that instead of using the `commit` middleware, we called a similar method on `request`. It's a more minimal version of the same capability.

Also notice `next` doesn't need to be called, since we are sure there is no additional middleware that needs to run.

This is an even more minimal middleware than the first example, but we aren't doing anything that necessitates it. Let's further customize to take advantage of this additional power:


```js
FOO: {
  path: '/foo/:flag',
  middleware: [
    api => async (request) => {
      const { api, actions, commit, params } = request
      if (params.flag) await commit()

      const payload = await api.fetch('something')
      actions.foo.complete(payload)

      if (!params.flag) await commit()
      analytics.track('complete', request)
    }
  ]
}
```

In essence, it's easier to take control of ordering with custom middleware.

You can also, specify some work that you only want to happen once at startup:


```js
middleware: [
  api => {
    const entryUrl = api.history.url            // only run once on app startup

    return async (req, next) => {               // called every time this route is matched
      await next()
      analytics.track('route', req, entryUrl)
    }
  },
  // ...
]
```

You can also call existing middleware *within a middleware*:

```js
middleware: [
  api => async (req, next) => {  
    await req.commit()                   
    await changePageTitle(api)(req, next)
  },
  // ...
]
```

If you wanted to do more work before calling the next middleware, you'd pass your own `noOp` next:

```js
thunk: () => ...,
middleware: [
  api => async (req, next) => {  
    await req.commit()    

    const noOp = () => Promise.resolve() // noOp means "no operation"              
    await changePageTitle(api)(req, noOp)

    doSomething()
    await next()
  },
  // ...
]
```

Or if the middleware was designed like `call`, you could simply leave out the `next` and a default `noOp` will be called internally:

```js
thunk: () => ...,
middleware: [
  api => async (req, next) => {        
    const returnValue = await call('thunk')(api)(req)
    await next()
    log(returnValue)
  },
  commit
]
```

We have studied the subject through customizing middleware at the route level, but these could easily be passed to `createApp` or `createModule`:

```js
import { call } from 'respond-framework/call'

const customCall = (name, config) => api => async (req, next) => {        
  const returnValue = await call(name, config)(api)(req)
  await next()
  log(name, returnValue)
  return returnValue
}

createModule(config, options, [
  customCall('beforeEnter'),
  commit,
  customCall('thunk', { cache: true })
])
```

And there you go, you've achieved mastery of customizing your middleware. See, that wasn't so bad. **MIDDLEWARE MIDDLEWARE MIDDLEWARE** is the name of the game with Respond Framework. 

*First Responders,* let's now delve into the built-in and provided middleware so you don't find yourself reinventing the wheel:


## `pathlessRoute` Middleware

Thanks to the `pathlessRoute('thunk')` middleware, "routes" in Respond don't need to have a `path` property. 

Pathless routes don't change the URL. They provide a familiar strategy to trigger some work, usually async work.

Pathless routes can be used for anything you want, but one primary purpose they serve is to "shadow" a *possible* route change. They let you setup and perform important async work before a route change can be determined. 

The primary use case is forms, where you must submit data, validate the response on the server, and only then *if successful* dispatch to a new route.  

Imagine you're on **`/draft/my-article-on-how-dope-respond-is`**:

```js
EDIT_DRAFT: {
  path: '/draft/:slug',
  thunk: ({ api }, { params }) => api.get(`draft/${params.slug}`)
},
```

Then you click **`<Button onClick={actions.submitDraft} />`**:

```js
SUBMIT_DRAFT: {
  thunk: async ({ api, state }) => {
    const { success, message } = await api.post('article', state.draft)
    return success ? actions.drafts() : actions.flash(message)
  }
},
```

If successfull, you end up on **`/my-drafts`**:

```js
DRAFTS: {
  path: '/my-drafts',
  thunk: ({ api }) => api.get('drafts')
},
```

Submitting forms therefore follows a precise pattern with Respond that mirrors links. Instead of buttons linking to pathful routes, pathless routes are used to submit the form data, and **on completion redirect to a pathful route.**


## `anonymousThunk` Middleware

The `anonymousThunk` middleware is an ode to *Redux Classic*. When using Respond, you almost never want to use it, but it is useful when you want to get something done from components--perhaps while prototyping--and you don't want to have to setup a route (or pathless route). 

For newcomers that missed the Redux era, it allows you to dispatch a `function` (usually an async one) in order to do some work within it and then dispatch a result in response to that work:

```js
const MyComponent = (props, state, actions) => {
  const onClick = actions.dispatch(async request => {
    const { api, dispatch } = request
    const state = await api.fetch('something')
    return actions.someRoute({ state })
  })

  return <Button onClick={onClick} />
}
```

As you can see, it provides information not available in your component via the `request` argument. You can use that to reach into your routes, access an `api` instance, etc.

> NOTE: If you use this, you will create a hole in the automatic tests Respond can generate for you. It will be able to track what happens once `someRoute` is called, but won't be able to get you any valuable test coverage for the code in the thunk function before that call. Therefore use this only for quick prototyping. See [testing](./testing.md) for more info.


## `call` Middleware

By now you've seen the `call` middleware countless times. By creating your own middleware above you got 
an idea what `call` might be doing. `call` is just a generalized version offering additional capabilities via the options you can pass to it:

```js
const name = 'thunk'

const config = {
  client: boolean = true,
  server: boolean = true,
  hydrate: boolean = true,
  shortcircuit: boolean = true,
  autoDispatch: boolean = false,
  prev: boolean = false,
  start: boolean = false,
  fallbackMode: boolean = false,
  shouldCall: request => [boolean, boolean],
  skip: 'options' | 'route' | 'none' = 'none',
  cache: boolean | Function = false,
  shortcuts: boolean = true
}

call(name, config)
```

Let's examine them one by one:


### `name` *(required)*

The `name` argument determines the name of the callback to call on route and options objects.


### `config.client` *(default: true)*

Toggle whether the callback runs client-side or not.


### `config.server` *(default: true)*

Toggle whether the callback runs server-side or not.


### `config.hydrate` *(default: true)*

Toggle whether the callback runs as the first route on the client. If, like `route.thunk`, its job is to get data that was already fetched on the server, you don't need to do it again. This configuration resolves that.


### `config.shortcircuit` *(default: true)*

Toggle whether the callback can short-circuit the middleware pipeline by returning false. 


### `config.autoDispatch` *(default: false)*

Enables returns of the callback to be dispatched with the `complete` action type for the given route. 

The return will be assigned to `action.payload`. `false` or `undefined` return values will not be dispatched. `null` will be dispatched. A return value of a string that is inferred as a route type will dispatch an action with that type.

If you manually dispatched in the callback already, another dispatch will not be performed for return values.


### `config.prev` *(default: false)*

Determine which route the callback should be called on. By default, it's the route being entered, but by setting `prev` to `true` you can call callbacks on the previous route, such as in `beforeLeave` and `onLeave` in the default middleware.


### `config.start` *(default: false)*

If you wanta data-fetching callback like `thunk` to run before the `commit` middleware, and you want the fetched data payload to be dispatched at the same time as `commit`, you're going to want to use this. In typical Respond fashion, `start` allows you to avoid more dispatches than necessary.

With `start` enabled a preemptive action is dispatched right before your async callback. Your reducers can listen to this action. It will have the type: `ROUTE_NAME.start`. 

#### example:

```js
const middlewares = [
  call('beforeThunk', { start: true }),
  commit,
]
```

The sequence for this middleware would be:

```js
// ROUTE_NAME.start dispatched
// route.beforeThunk called
// ROUTE_NAME dispatched with payload returned from beforeThunk
```

You may not even need to listen to it in your reducers, as `state.location.ready` listens to it and will be set to `false`. This allows you to display loading spinners just as you regularly would.

> Using this option is a drastic change to how your app runs. In essence with this option enabled, the URL and application state doesn't change until you have the data you need. 

The `ROUTE_NAME.start` action has all the same info as dispatched on `commit`. Therefore your reducers *could* treat this action as if it's the ultimate state change action. **That's not its use case though**; its use case is to trigger loading indicators. For example, Respond's built-in location reducer doesn't change until commit.

Use this option if:

- your app authenticates asynchronously every route, and you don't want to flash glimpses of a blocked route
- you prefer not to see URL changes in the address bar until route transitions are finalized

> With the standard setup, you would have to dispatch `actions.back` after commit, resulting in jank on the page (and in the address bar).

You can also put additional middleware before commit. For example the following allows you to naturally break up authentication related async work into `beforeEnter` and data-fetching async work into `thunk`:

```js
const middlewares = [
  call('beforeEnter', { start: true }),
  call('thunk', { cache: true, autoDispatch: true })
  commit,
]

ROUTE_NAME: {
  path: '/route',
  beforeEnter: async ({ api, state }) => {
    const isAllowed = await api.authenticate(state.user)
    return isAllowed
  },
  thunk: ({ api }) => api.fetch('data'),
}
```

Respond is smart enough to attach the payload of the closest callback to the `commit` action, even though it doesn't have the `start` option. Therefore visiting the above route will result in the following sequence:


```js
// 1. beforeEnter middleware called, and this action dispatched:
{
  type: 'ROUTE_NAME.start' // aka types.routeName.start
  pathname: '/route',
  // etc
}

// 2. beforeEnter callback called
// 3. thunk callback called (if beforeEnter did not return false)

// 4. commit middleware executed, and this action dispatched:
{
  type: 'ROUTE_NAME' // aka types.routeName
  pathname: '/route',
  // etc
  payload: fetchedData // reducers now look for payload in ROUTE_NAME instead of ROUTE_NAME.complete
}
```

But to make this work `autoDispatch` must be enabled on the subsequent middleware. The `start` option works in conjunction with `autoDispatch` and is automatically enabled for the middleware that has `start` enabled. However for other middleware, you must manually enable `autoDispatch`. 

Accordingly, manual dispatches are designated as an escape hatch from the standard sequence. 






### `config.fallbackMode` *(default: false)*

Enable to skip options level callbacks when the route level callback is available. In other words, call one, not both, prioritizing the callback attached to the route.


### `config.shouldCall`

Optionally you can provide a function which manually determines if  callbacks should be called. It's passed the `request` object, and returns a tuple containing 2 values. 

For example the following allows routes to prevent themselves from being called:

```js
config.shouldCall = request => request.route.skip
  ? [true, false]
  : [true, true ]
```

### `config.skip` *(default: 'none')*

Set to `'options'` to skip options level callbacks, and set to `'route'` to not call routes level callbacks. 



### `config.cache` *(default: false)*

Enable caching for the given callback. If enabled, callbacks won't be called multiple times, and it will be assumed the necessary state is already in the store from the first call. By default, cache hits are determined by the full URL string minus the hash:

```js
call('thunk', { cache: true })
```

*is equivalent to:*

```js
call('thunk', { cache: (action, name) => {
  const { type, basename, pathname, search } = action
  return `${name}|${type}|${basename}|${pathname}|${search}` 
}})
```

This is an important topic with several ways to customize the created cache key and several ways to invalidate the cache. 

See [caching](./caching.md) for more information.


### `config.shortcuts` *(default: true)*

By default, callbacks can be in several forms 

- `route.callback = () => ...`
- `route.callback = [() => ..., () => ..., etc]`
- `route.callback = 'ANOTHER_ROUTE'`
- `route.inherit = true`
- `route.callback = ['ANOTHER_ROUTE', () => ..., mixNmatch]`

Set to `false` to disable all but the function form.

See [Advanced Route Config](./advanced-route-config.md) for more info



## `commit` Middleware

The `commit` middleware isn't customizable, but it's the only required middleware that you have to place somewhere in your middleware array yourself. This allows you to determine exactly what happens before and after the state and URL changes are committed. 


## `transformAction` Middleware

`transformAction` is the most important middleware of them all, but also one operating completely transparently behind the scenes. 

What it does is generate the fully flushed out routing action that is passed to your reducers and your callbacks. It's built-in, so the only way to customize it is through the [`compose` escape hatch](./advanced-options.md).

Unless you're fixing a bug or adding new capabilities, you likely will never have to concern yourself with this one. Just be aware of how it works: 

- *it takes the plain actions initially dispatched in response to UI events (which minimally have a `type`)*
- *and then cross-references them against your routes + existing state in order to tack on all the additional juicy information you could dream of*
- that way your reducers and callbacks don't need to concern themselves with common transformations and logic


## `changePageTitle` Middleware

`changePageTitle` is a very simple middleware to set the page title client side on route transitions. It receives one optiona argument that accepts a string:

```js
changePageTitle('title')
```

for the name of the slice of state that contains the string you want to use for the title. 

or a selector function to manually select the title from state: 

```js
changePageTitle(state => state.title)
```

The default is to look for a reducer named `title`.

The purpose of this middleware is to be minimal and inspire you. You don't have to store titles in state. It's an approach we recommend because it allows you to re-use titles that you're already using within components. It also makes it easy to extract from state as part of server-side rendering. 

Another approach would be to add a `title` key to each of your routes, which uses a function to return the title.

Lastly, you likely will want to change other meta tags in your `<head>`. Help contribute to Respond by making a more comprehensive middleware. You can borrow from how [Helmet](https://github.com/nfl/react-helmet/blob/master/src/HelmetUtils.js) does it at the component level. PR a link to your package in place of this paragraph when you do :)


## `serverRedirect` Middleware

`serverRedirect` is specialized middleware whose purpose is to short-circuit the middleware pipeline early on the server when a redirect is detected, thereby saving your servers from wasting cycles. 

You can use the `isRedirect` utility to efficiently detect if the action generated by Respond was a redirect, and then perform the necessary work to immediately bail out:

```js
import { isRedirect } from 'respond-framework/utils'

export default async configureServerStore(req, res) {
  // ... 

  const { store, firstRoute } = createApp(config, options)
  const action = await store.dispatch(firstRoute())

  if (isRedirect(action)) {
    const { status, url } = action
    res.redirect(status || 302, url) // short-circuit because request is complete
    return false
  }

  // ...
  return store
}
```
