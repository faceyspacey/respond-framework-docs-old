# Middleware

The crown jewel of Respond framework is its middleware pipeline. There's endless things you can do with it. It greatly simplifies your app, providing some much needed best practices and architecture. 

The pipeline linearly consolidates related work that otherwise ends up confusingly sprinkled throughout your component tree. 

It will provide you great developer happiness if any of the following are true:

- You're tired of pulling your hair out wondering where xyz is happening in your component tree.
- You're tired of surprises surfacing as components render, and desire more predictability.
- You remember the server-side MVC days and you seek a return to normalcy.
- You recognize the stability provided by sequential ordering of effects as in traditional software, and you recognize it's about time the pendulum swings back into balance for Reactlandia.
- **You understand that the React community was in fact wrong that all data dependencies *cannot be known* simply by the top level URL/route. They in fact *can*, especially with a routing centric framework such as Respond. *Therefore it makes no logical sense to further complicate your lives by splitting up such work across a component tree of unknown (and very likely deep) depth*.**


Regarding the last point, there are very few apps that can't determine their data dependencies based on a URL. The only requirement is that you URL-ize every state your app can be in, which Respond makes frictionless. In other words, provide a *"uniform resource indicator,"* which makes a lot of sense to do anyway, and comes with many additional benefits. 

> If you're wondering about the enormous number of states your app can be in, keep in mind that with Respond your app will have far fewer states, and instead appropriately *respond* to the 1-2 actions dispatched per route transition.

Customizing your middleware is how you bring conventions and architecture to your app, while greatly simplifying it. Let's take a look:



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
  changePageTitle,
  call('onLeave', { prev: true }),
  call('onEnter'),
  call('thunk', { cache: true }),
  call('onComplete')
]
```

You can customize the middleware used via the 3rd argument to `createApp`:

```js
const middlewares = [
  call('beforeEnter'),
  commit,
  call('onEnter' { cache: true })
]

createApp(config, options, middlewares)
```

Each module will use its own middleware if specified and ignored the middleware of parents:

```js
createModule(config, options, middlewares)
```


With the given usages of the `call` middleware, you can expect to have available matching callbacks on your routes:

```js
routes: {
  FOO: {
    path: '/foo',
    beforeEnter: () => ...,
    onEnter: () => ...,
  }
}
```

> This is where the conventions and architecture come into play. This is the default convention Respond prescribes.



You can also override the middleware used at the route level:

```js
routes: {
  FOO: {
    path: '/foo',
    middleware: [...]
  }
}
```



## Built-in Middleware

The built-in middleware is prepended even if you provide your own custom middleware array:

```js
middlewares: [
  serverRedirect,      
  anonymousThunk,
  pathlessRoute('thunk'),
  transformAction,
  // ...whatever you specify
```

Therefore, if you specified the above custom middleware array, you actually get this:

```diff
middlewares: [
+  serverRedirect,      
+  anonymousThunk,
+  pathlessRoute('thunk'),
+  transformAction,
  call('beforeEnter'),
  commit,
  call('onEnter' { cache: true })
]
```

That's nothing you need to concern yourself with. It's just Respond keeping a **"slim core"** while re-using its own plugin API (i.e. a hallmark of good software design).

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

Let's take a look at the first point:

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

Notice that instead of using the `commit` middleware, we called a similar method on `request`. It's a more minimal version of the same capability. Only use it if you know what you're doing. 

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

      if (!params.flag) await.commit()
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

    return (req, next) => {                     // called every time this route is matched
      analytics.track('route', req, entryUrl)
      return next()
    }
  }
]
```


## Pathless Routes

"Routes" in Respond don't need to have a `path` property and result in a URL changing in the address bar. Pathless routes serve the purpose to "shadow" a real route changing route. They let you setup and perform important async work before a route change can be determined. The primary use case is forms, where you must submit data, validate the response on the server, and only then dispatch to a new route.  

Of course you can use them for any async needs you have like thunks, sagas, or epics. What they provide, in conjunction with regular routes, is a uniform structure, aka *"convention."* Seeing related routes (of both kinds) grouped together in a flat routes map, as simple as it is, has been something missing from Redux development. 




