# Respond Framework Walkthrough


![Respond Framework Homepage](./docs/images/respond-homepage.png)

Respond Framework is what happens if you build Redux & first-class concerns for routing into React, plus take a page from the traditional server-side MVC playbook when it comes to side-effects. 


Here's a quick overview of the features and usage in Respond Framework.



## Installation

```sh
yarn add 'respond-framework'
```


## "Redux Components"

For a taste of where we're going, behold:

```js
function LoginComponent(props, state, actions) => {
  const onClick = state.session ?  actions.logout : actions.login
  const text = state.session ? 'LOGOUT' : 'LOGIN'

  return (
    <div>
      <button onClick={actions.login}>{text}</button>
    </div>
  )
}
```

Gone are the days of `connect` + `mapState/DispatchToProps`. Had the additional args been left out, Babel would have compiled no alterations under the hood.

More on this later...



## Modular (did I hear you say "Redux Modules"??)

Respond Framework is modular and encapsulated like React components, but at a higher level. It gives you a birds-eye perspective of important portions of your app & enables separate developer teams to go off and build with the confidence that what they bring back will plug in nicely.

Your app is composed of *Respond Modules*, which encapsulate **routes**, **components**, **reducers** and everything else you may need. `moduleProps` allow you to share `state`, `actions`, and `types` between parent modules and their children.


*src/modules/app/index.js*

```js
import { createApp } from 'respond-framework'
import ReactDOM from 'react-dom'
import enhancer from './enhancer'
import reducer from './reducer'
import components from './components'

const { firstRoute, store } = createApp({
  enhancer,
  reducer,
  initialState: window.RESPOND_STATE,
  components,
  routes: {
    home: '/',
    login: '/login',
    dashboard: {
      path: '/dashboard',
      load: () => import('../modules/dashboard'),
      moduleProps: {
        state: {
          user: 'session'
        },
        actions: {
          login: 'login'
        }
      }
    }
  }
})

(async function() {
  await store.dispatch(firstRoute())

  ReactDOM.hydrate(
    <Provider store={store}>
      <App />
    </Provider>,
    document.getElementById('root')
  )
})()
```


*src/modules/dashboard/index.js:*

```js
import { createModule } from 'respond-framework'
import { session } from './reducer'
import { Dash, Metrics, Stats } from './components'

export default createModule({
  reducers: { foo, bar },
  components: { Dash, Metrics, Stats },
  routes: {
    main: '/',
    metrics: '/metrics',
    stats: '/stats'
  }
})
```

> the configuration passed to `createApp` is also a module. *Respond* is "modules" all the way down.


## Predictable Linear Effects

No longer leave side-effects up to random discovery in your component tree!  

> ["No surprises == better sleep"](https://twitter.com/faceyspacey/status/1107057805507227649) -Anton Korzunov (maintainer of React-Hot-Loader, react-imported-component)

Instead orchestrate them linearly once per route. 

Trust us, React is great--**why do you think** ***Respond*** **is built on top of it**--but that doesn't mean every approach the React team promotes is spot on. Side-effects don't belong in your components, even with hooks :)


### Here's how we roll:


```js
export default createModule({
  reducers,
  components,
  routes: {
    main: {
+      path: '/',
+      thunk: ({ api }) => api.fetch('items')
    },
    metrics: '/metrics',
    stats: '/stats'
  }
})
```

### Or if you're old school:

```js
export default createModule({
  reducers,
  components,
  routes: {
    main: {
      path: '/',
+      thunk: async ({ api, dispatch, types }) => {
+        const payload = await api.fetch('items')
+        dispatch({ type: types.main.COMPLETE, payload })
      }
    },
    metrics: '/metrics',
    stats: '/stats'
  }
})
```
> `types`, `actions`, `state`  supplied via dependency injection to avoid conflicts; which also means less importing

### Add callbacks that fire for all routes:


```js
import mixpanel from 'mixpanel'

export default createModule({
  reducers,
  components,
  routes: {
    main: {
      path: '/',
      thunk: thunk: ({ api }) => api.fetch('items')
    },
    metrics: '/metrics',
    stats: '/stats'
  }
}, {
+  onEnter: ({ location }) => mixpanel.track('dash', location)
})
```

> they are run in parallel to route-specific ones

### Bail out using redirects before route changes:



```js
export default createModule({
  reducers,
  components,
  routes: {
    main: {
      path: '/',
      thunk: thunk: ({ api }) => api.fetch('items')
    },
    metrics: '/metrics',
    stats: '/stats'
  }
}, {
+  beforeEnter: async ({ getState, actions }) => {
+    if (!getState().session) {
+      return actions.login() // remember, the login action was passed as a `moduleProp`
+    }
+  },
  onEnter: ({ location }) => mixpanel.track('dash', location)
})
```

> `beforeEnter` could just as easily been added to individual routes


### Bail out on leave:



```js
export default createModule({
+  reducers: { session, acceptedCookies },
  components,
  routes: {
    main: {
      path: '/',
      thunk: ({ api }) => api.fetch('items'),
+      onLeave: ({ getState }) => !getState().acceptedCookies // return false to block route change
    },
    metrics: '/metrics',
    stats: '/stats'
  }
}, {
  beforeEnter: async ({ getState, actions }) => {
    if (!getState().session) {
      return actions.login()
    }
  },
  onEnter: ({ location }) => mixpanel.track('dash', location)
})
```


## 100% Customizable Middleware

The backbone of Respond is our routing slash side-effects library, *Rudy*, which is hard at work under the hood. *Rudy* offers an async middleware API similar to [koa-compose](https://github.com/koajs/compose) with "rewind."

What this means is that each middleware asynchronously gets a chance to operate on the `route` before passing the request to the next middleware in the chain. 

> classic Redux offers a synchronous middleware API, ours is async to meet the demands of today.

The above `routes` could be minimally served with this custom middleware pipeline:

```js
export default createModule(config, options, [
+  transformAction, 
+  call('beforeEnter'),
+  enter,
+  call('onLeave', { prev: true }),
+  call('onEnter'),
+  call('thunk', { cache: true }),
])
```

All the callback names passed to `call` are available as keys on your routes and executed at the appropriate time during route transitions. The `call` middleware has many other super powers like automatic dispatch of returns.

Each middleware also has a 2nd chance to peform work as the chain "rewinds." This gives us great control over route transitions. 

For example, we can bail out at any time (before or after `enter`). We have even figured out how to do this in bail-outs triggered by browser back/next buttons, thanks to our custom `History` package within core. *By the way, our `History` is truly one of a kind--first in the world to keep track of browser history `entries`; more on that to come...*

> If you're wondering, yes, you can still use the traditional Redux enhancer/middleware APIs like the Devtools and Sagas



## Generated Action Creators & Types

Your `routes` object generates all the action types your application needs. For example in the current app, so far we have:

- `actions.home()`
- `actions.login()`
- `actions.dashboard()`
- `actions.dashboard.metrics()`
- `actions.dashboard.stats()`

As well as the following for each route:

- `actions.home.complete()`
- `actions.home.error()`
- `actions.dashboard.metrics.complete()`
- etc


The types generated are just the capitalized snake_cased versions of these prefixed by their namespaces:

- `HOME`
- `LOGIN`
- `DASHBOARD`
- `DASHBOARD/METRICS`
- `DASHBOARD/STATS`

And:

- `HOME_COMPLETE`
- `HOME_ERROR`
- `DASHBOARD/METRICS_COMPLETE`
- etc

They are accessible in their lowercased form at, for example: `types.home` or `actions.home.type`.

Along with `state`, they are **injected** into callbacks:

```js
thunk: ({ types, actions, state }) => 
```

Into reducers:

```js
reducers: {
  foo: (state, action, types) => types.home ? true : state
}
```

And components:

```js
const LoginButton = (props, state, actions) => {
  return (
    <div>
      <button onClick={actions.login}>LOGIN</button>
    </div>
  )
}
```


Because *Respond Modules* are guaranteed to be unaware of the outside world **(even though they're conveniently using the same store)**, `actions`, `types` and `state` must be injected by the framework. This allows Respond to transparently normalize namespace access under the hood via proxies, so you only have to use namespaces where you absolutely must, which brings up an important point:

**Actions, state and types from child modules is available in parent components by their namespace.** Whereas child modules must use `moduleProps` to access the same from the parent. In other words, parents get to know whats up with their children, but not the other way around *(kind of like in real life :)*



## Automatic Code Splitting

Respond automatically code splits your app. There's nothing you have to do about it, it just happens through normal use of modules. 

It's provided through one of our middleware. If you don't provide a `middleware` array, here's the default one:

```js
export default createModule(config, options, [
  serverRedirect,           // short-circuiting middleware       
  anonymousThunk,
  pathlessRoute('thunk')   
  transformAction,          // pipeline starts here
>  codeSplitModule('load'),  
  call('beforeLeave', { prev: true }),
  call('beforeEnter'),
  enter,
  changePageTitle,
  call('onLeave', { prev: true }),
  call('onEnter'),
  call('thunk', { cache: true }),
  call('onComplete')
])
```

The `codeSplitModule` middleware is responsible for insuring modules (including their components, reducers and route side-effects) are loaded before the route executes. **In other words, the plane is built while flying.**

Some middleware--**the "short-circuiting" middleware**--serves to perform a quick function before quickly exiting the pipeline. *Example:* handling anonymous thunks, or routes that have just a thunk but no path (which is a convenient way to consistently use the same route configuration object API).

## Seamless Prefetching


Routes can also be prefetched, including both their Webpack chunks and data dependencies which get cached.

**To prefetch** potential subsequent routes, return a list of them from the current route using the `prefetch` handler:

```js
routes: {
  login: '/login',
  home: {
    path: '/',
    prefetch: ({ actions, types, getState }) => [
      types.login,
      actions.profile({ params: { id: getState().user.slug }})
    ]
  },
  profile: {
    path: '/profile/:slug',
    thunk: ({ api, params }) => api.fetch(`users/${params.slug}`)
  }
}
```

If you provide a precise action, both the chunk *and callbacks such as thunks* will be called (with their results **cached**, aka stored in Redux).

If you, supply just a string as in `types.login`, only the chunk for the matching route will be called (since thunks wouldn't know what to fetch without precise params/etc).


## Serve Chunks w/ SSR

SSR is challenging. Code Splitting is challenging.

SSR + Splitting unfortunately is greater than the sum of its parts, which is to say ***combination SSR + Splitting*** **is many times more challenging.** 

With Respond it's *just* a matter of passing the `request` `url` and *awaiting* your `firstRoute()`. 


*server/configureStore.js:*
```js
import { createApp } from 'respond-framework'

export default async function configureStore(request) {
  const options = {
    initialEntries: [request.url]
  }

  const { firstRoute, store } = createApp(config, options)

  await store.dispatch(firstRoute())

  return store
}
```

And then extracting used chunks from state:

*server/serverRender.js:*
```javascript
import ReactDOM from 'react-dom/server'
import { Provider } from 'respond-framework'
import configureStore from './configureStore'
import App from '../src/components/App'

export default async function serverRender(req, res) {
  const store = await configureStore(req)

  const state = store.getState()
  const appString = ReactDOM.renderToString(<Provider store={store}><App /></Provider>)
  const stateJson = JSON.stringify(state)

  // like this:

  const { chunks } = state.location

  const stylesheets = chunks.map(chunk => {
    return `<link rel='stylesheet' href='/static/${chunk}.css' />`
  }).join(' ')

  const scripts = chunks.map(chunk => {
    return `<script src='/static/${chunk}.js' />`
  }).join(' ')

  return res.send(
    `<!doctype html>
      <html>
        <head>
          ${stylesheets}
        </head>
        <body>
          <div id="root">${appString}</div>
          <script>window.RESPOND_STATE = ${stateJson}</script>
          <script src="/static/bootstrap.js" />
          <script src="/static/vendors.js" />
          ${scripts}
        </body>
      </html>`
  )
}
```

*server/index.js:*
```js
import express from 'express'
import serverRender from './serverRender'

const app = express()
app.get('*', serverRender)
http.createServer(app).listen(3000)
```

SSR with *Respond* doesn't lock you into a *"walled garden"* (like other solutions) for how to manage your application's server. You can use Express, Koa, Node, Babel and Webpack as you normally would.


Yes, we wrote the book when it comes to routing, splitting and SSR, especially in a Redux world. ***Respond Framework*** **is the direct heir to:**

- **[faceyspacey/redux-first-router](https://github.com/faceyspacey/redux-first-router)** and 
- **[faceyspacey/react-universal-component](https://github.com/faceyspacey/react-universal-component)**



## Baked-in Redux (check out our sweet Components!)

Internally, our state management library is called *Remixx*, but it's ok if you continue to call it *"Redux"* :)

"Redux Modules"--*you know the ones the community never figured out how to make*--was always about components. The idea is that you can provide a pairing of Redux assets (reducers, actions, types) and components in a format that you can share with 3rd parties without naming conflicts (such as through NPM). In other words: **mini apps**. *It would have been nice, but when was the last time you saw Redux-based components on NPM??* **Not usefully now.**

While making this possible, we took the liberty to build in exactly what you might expect into React. Here's what *Respond components* look like:

```js
const LoginButton = (props, state, actions) => !state.session && <Button onClick={actions.login} />
```

**`state` and `actions` are passed as arguments in addition to props!** 

Actions are automatically bound to `dispatch` and there's never any need for `mapStateToProps`. Under the hood proxies are used to track your actual usage of state so re-renderings only occur if the precise nested piece of state you accessed has changed. *This applies both to reducers and selectors. More on selectors below.*

> Yes, *Respond* is built for the era where you assume your users' browsers support proxies.

The transformation of your components to support this interface is done via Babel, therefore if you don't happen to use `state` or `actions`, your components will be left untouched. You're free to use hooks, side effects, you name it (though we recommend keeping your side-effects in *Respond* routes, especially if you're doing SSR).


## Automatic Namespacing

If you saw:

```js
import { Link, createModule } from 'respond-framework'

const MetricsButton = (props, state, actions) => {
  const text = state.foo ? 'Go To Metrics' : 'metrics' // contrived example

  return (
    <div>
      <Link to={actions.metrics}>{text}</Link>
    </div>
  )
}

export default createModule({
  reducers: { foo, bar },
  components: {
    Dash: (props) => {
      return (
        <div>
          <MetricsButton />
        </div>
      )
    }
  },
  routes: {
    main: '/',
    metrics: '/metrics',
    stats: '/stats'
  }
})
```

and were wondering how `actions.metrics` and `state.foo` was guaranteed to be unique if this component was part of a module on NPM, *you'd be a keen observer.*

Under the hood (within the `state` and `actions` proxies) here's what's actually being called:

```js
const MetricsButton = (props, state, actions) => {
  const text = state.dashboard.foo ? 'Go To Metrics' : 'metrics'

  return (
    <div>
      <Link to={actions.dashboard.metrics}>{text}</Link>
    </div>
  )
}
```

That's because the component knows what module it's part of. It doesn't need to provide its own namespace. It wouldn't even work if it tried. *Respond components* have no awareness of the outside world *unless it's told about it.*

Like import aliasing in ES6 modules, the namespace is *assigned in the parent module*. Remember this:

*src/modules/app:*

```js
dashboad: {
  path: '/dashboard',
  load: () => import('../modules/dashboard'),
}
```

> The parent route type, `DASHBOARD` doubles as the module's namespace for nested routes




## Module Props

Modules can receive `props` just like components can. The purpose is different though: 

- **`moduleProps` serve the purpose of aliasing existing state coming from parent modules**
- **and therefore keep all state in a** ***single easily time-travellable + testable store*** 

Here's how you tell the `dashboard` module about pre-existing state and actions in the parent:

```js
dashboad: {
  path: '/dashboard',
  load: () => import('../modules/dashboard'),
+  moduleProps: {
+    state: {
+      user: 'session'
+    },
+    actions: {
+      login: 'login'
+    },
+    types: {
+      CLOSE: 'HOME'
+    }
+  }
}
```

> So `getState().user` will be made available through the `state` proxy at `state.session` and the equivalent for `actions.login` in the `actions` proxy. It's just coincidence that the child module documents an expected action by the same name of `login`.


## Module Parameterization 

At the end of the day component `props` are just parameters. Our modules also can be parameterized. The difference from `moduleProps` is that there is no special behavior under the hood. 

If you wanted to choose paths used in a child module from the parent, here's how you would do it:


```js
import { createModule } from 'respond-framework'

export default createModule((options) => ({
  routes: {
    OPEN_CART: {
      // path: '/cart',
      path: options.openCartPath, // yes, groundbreaking
      thunk: ({ stripe, payload }) => stripe.findCartItems(payload)
    },
    CHARGE: {},
    CONFIRMATION: {}
  }
}))
```

*parent module/app:*

```js
CHECKOUT: {
  load: async () => {
    const module = await import('respond-stripe-cart')
    return module({ openCartPath: '/all-i-do/is-win' })
  }
}
```

*Batteries not included*


## Modules, Nesting, Splitting, Oh My!

The cornerstone of Respond's pristine interface is one thing: **the collapsing of many capabilities under one interface:** ***Respond Modules.*** 

Let's take a look at how dynamically imported routes look like after being merged:


**before:**
```js
// parent module:
routes: {
  dashboard: {
    path: '/dashboard',
    thunk: ({ api }) => api.fetch('user'),
    load: () => import('../modules/dashboard'),
  }
}

// child dashboard module:
routes: {
  metrics: '/metrics',
  stats: '/stats'
}
```

**after:**
```js
routes: {
  dashboard: {
    path: '/dashboard',
    thunk: ({ api }) => api.fetch('user'),
    routes: {
      metrics: '/metrics',  // reified path /dashboard/metrics
      stats: '/stats'       // reified path /dashboard/stats
    }
  }
}
```

As you can see, modules, path nesting, and code splitting are all **collapsed** into a single unified interface.

But that's not all, the `thunk` above has a special characterstic we call **"callback nesting"**: 

- it's called even if you visit `/dashboard/metrics` directly
- it's not called if you navigate from `/dashboard/metrics` to `/dashboard/stats` though

It's only called on first entrance of the given group of nested routes/module. The common use case for this is to insure that the `user` or `session` object exists for all dashboard routes, without having to code the call for every route.


Let's check out a few more scenarios:



### Use a different path prefix than the parent:

```js
routes: {
  dashboard: {
    path: '/dashboard',
    pathPrefix: '/something-else',
    thunk: ({ api }) => api.fetch('user'),
    routes: {
      metrics: '/metrics',  // reified path /something-else/metrics
      stats: '/stats'       // reified path /something-else/stats
    }
  }
}
```


### Leaving out the parent module path:

```js
routes: {
  dashboard: {
    thunk: ({ api }) => api.fetch('user'),
    routes: {
      metrics: '/metrics',  // reified path /metrics
      stats: '/stats'       // reified path /stats
    }
  }
}
```

> In essence, the route nesting is being used just for callback nesting + module namespacing; usually you can visit: `actions.dashboard()`, since the namespace doubles as its own route; in this case you can't. 



### Leaving out the parent module path:

```js
routes: {
  dashboard: {
    path: '/dashboard',
    pathPrefix: false,
    thunk: ({ api }) => api.fetch('user'),
    routes: {
      metrics: '/metrics',  // reified path /metrics
      stats: '/stats'       // reified path /stats
    }
  }
}
```

> This allows you to dispatch `actions.dashboard()` and reach an independent route as you originally could, while allowing for child routes to be unprefixed. It's a hybrid of the previous ones.


### Merge single child route into parent:

```js
// pre-import:
dashboard: {
  path: '/dashboard',
  load: () => import('../modules/dashboard'),
}

// dashboard module:
routes: {
  entry: {
    path: '/foo/:param',
    thunk: ({ api }) => api.fetch('user'),
  }
}

//post-import:
routes: {
  dashboard: {
    path: '/dashboard/foo/:param',
    thunk: ({ api }) => api.fetch('user'),
  }
}
```

> If a child route is named `entry` it will be merged into the parent, and it's path appended to the parent's. Within itself the child must refer to this action as `entry` obviously. 

> NOTE: no additional routes at the same level as `entry` are allowed. You can nest them further though.


## Reducers & Selectors

In *Respond*, selectors are also passed at time of store creation. In addition, both reducers and selectors are automatically combined.

```js
export default createModule({
  reducers: { foo, bar },
  selectors: {
    visibleCurrentItems: (state, before, after) => {
      const items = state.items[state.filter]
      return items.filter(item => item.date < before && item.date > after)
    }
  },
  etc
})
```

What gives?

The difference between reducers and selectors is:

- selectors have access to all those isolated reducers *and* other selectors. 
- selectors are often intended to be used with arguments from components (i.e. `props`)

Which is why it makes a whole lot of sense, in a world where you're no longer doing the `mapStateToProps` rigamarole, to have functions on your `state` object, a la:

```js
const ItemsList = (props, state) => {
  const items = state.visibleCurrentItems(props.before, props.after)

  return (
    <>
      {items.map(item => (
        <div>
          <h1>{item.title}</h1>
          <p>{item.blurb}</p>}
        </div>
      ))}
    </>
  )
}
```


> As far as perf, we automated render trigger/blocking like with reducers, using a combination of usage tracking + caching. Unlike the days of *reselect*, a single selector automatically works across multiple components. Think of your `state` object, containing both reducers + selectors, as your **UI Database**.




## Action + Reducer Shape

Having learned from [redux-first-router](https://github.com/faceyspacey/redux-first-router), *Respond* route actions and `location` state contain the richest set of info we could dream up. Our entire wishlist has been met.

Actions look like this:

Location State stored in *Remixx* looks like this:


## Route Options


The following are options that you can attach to each route, such as at `route.toPath => () => ...`

Unless indicated by an *astersik* (\*) you can also expect them to work within the `options` argument passed to `createModule`. The difference is that *module level options* work for all routes in the module.

### URL Transformations
| name | arguments | return |
| --- | --- | --- |
| toPath | arg1, arg2| return foo|
| fromPath | arg1, arg2| return foo|
| toSearch | arg1, arg2| return foo|
| fromSearch | arg1, arg2| return foo|
| toHash | arg1, arg2| return foo|
| fromHash | arg1, arg2| return foo|
| toState* | arg1, arg2| return foo|
| fromState* | arg1, arg2| return foo|

> Browser's allow a `state` object to be attached to each entry in the browser's `history` stack, that way when you revisit a URL, you have access to anything you want remembered. *Respond* makes it trivial to store reactive objects here, similar to state attached to components.

### Automatic Transformations
| name | type | default |
| --- | --- | --- |
| capitalizedWords | boolean | `false` |
| convertNumbers | boolean | `false` |


### Default Values
| name | arguments | return |
| --- | --- | --- |
| defaultParams | arg1, arg2| return foo|
| defaultQuery | arg1, arg2| return foo|
| defaultHash | arg1, arg2| return foo|
| defaultState | arg1, arg2| return foo|



### MISC
| name | arguments | return |
| --- | --- | --- |
| praseSearch | arg1, arg2| return foo|
| stringifyQuery | arg1, arg2| return foo|
| defaultHash | arg1, arg2| return foo|
| defaultState | arg1, arg2| return foo|


### Matchers
| name | arguments | return |
| --- | --- | --- |
| path | arg1, arg2| return foo|
| query | arg1, arg2| return foo|
| hash | arg1, arg2| return foo|
| fromQuery | arg1, arg2| return foo|
| toHash | arg1, arg2| return foo|
| fromHash | arg1, arg2| return foo|
| toState* | arg1, arg2| return foo|
| fromState* | arg1, arg2| return foo|

- transformation
- tons of info in state
- automatic basename handling



This is the mental model of components in *Respond*. But is each component truly *pure*? No, but you don't have to concern yourself with that. It is pure from your perspective when all you truly care about is purity of `f` aka `render`.



## Routing Components
- <Route />
- <Link />


## Misc Features
- History Entries Sync (world first!)


## Middlewares
- Caching
- pathless routes
- anonymous thunks




## Big Picture

Modularity + Linear Side Effects (copy monologue from beginning of pitch doc)




## Conclusion

So there you go--long standing ecosystem problems finally solved and just about every effective React concept consolidated/collapsed under one beautiful interface. GraphQL/Apollo middleware coming soon. Welcome to the **Rails of React**.
