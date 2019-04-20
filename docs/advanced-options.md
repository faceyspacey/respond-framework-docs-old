# Advanced Options

In this doc you will you learn to make powerful customizations of the Respond Framework for your app. Only use these if you know what you're doing and you're sure Respond can't handle your needs via built-in mechanisms.

Offering the core available for extension may seem controversial, but the result is not too different than extending classes like `ActiveRecord` from Rails.

We're able to do so because we've seen how most modern reactive apps are being built, and we're confident our core won't change so much that it's problematic to both of us.

Forking tools greatly reduces developer productivity. Now you likely don't need to go that route. 

The following customizations are to be *enjoyed* when you're in pain and you need to do something your unique app requires:


## `createReducer(initialState, routes): reducer`

`createReducer` will be called with the inital state and your routes object. Return a reducer of shape `(state, action, types) => nextState` in order to customize how your location state is handled. A common pattern is to use the built-in one and make minor changes, eg:

```js
import { createReducer } from 'respond-framework/core'

const options = {
  createReducer: (initialState, routes) => {
    const reducer = createReducer(initialState, routes)

    return (state, action, types) => {
      if (action.someFlag) {
        return {
          ...state,
          ready: action.someFlag
        }
      }

      return reducer(state, action, types)
    }
  }
}

createModule(routes, options)
```


## `createInitialState(action): state`

This function will receive the same action returned from `firstRoute` and generate the initial state from it. That initial state is the same state passed to `createReducer` above. The same pattern as with `createReducer` can be used here. Rarely will you need to fully customize this method.




## `createRequest(action, api, next): request`

Before the Respond pipeline can pass your callbacks a `request`, it has to create it. `createRequest` does nothing more than instantiate our `Request` class, which you are also free to extend. 

Here's how you can customize:

```js
import { Request } from 'respond-framework/core'

class AppRequest extends Request {
  performSpecialFeature() {
    if (this.getKind() === 'replace') console.log('foo')
  }
}

const options.createRequest = (action, api, next) => {
  return new AppRequest({ ...action, yourFlag: true }, api, next)
}
```

Now you can access `performSpecialFeature` in your callbacks like so:

```js
MY_ROUTE: {
  onEnter: ({ performSpecialFeature }) => performSpecialFeature()
}
```

Note that `performSpecialFeature` is pre-bound to `this` even though you destructured it off the `request` instance, similar to how you often do in class components.



## `createHistory(routes options): History`

Perhaps the only use case for this is fixing bugs in our internal `History` class, and you want to try your fixes out without forking. The internal one simply detects if the `BrowserHistory` class should be used or the `MemoryHistory` class:

```js
import { BrowserHistory, MemoryHistory, supportsHistory, supportsDom } from 'response-framework/history'

options.createHistory = (routes, opts = {}) =>
  supportsDom() && supportsHistory() && opts.testBrowser !== false
    ? new BrowserHistory(routes, opts)
    : new MemoryHistory(routes, opts)
```

Both classes extend `History`. Check `/src/core/history` for these classes.


## `createCacheKey(action, name): string`

This function creates the string by which the current callback will be cached. Keep in mind, we don't actually cache any data. It's assumed you do it in Redux/Remixx. However, if before a callback is called, we find a matching cache key, the callback to get the data won't be called. Keep in mind this only applies to middleware with the `cache` option true:

```js
call('thunk', { cache: true }),
```

Here's the default `createCacheKey` for you to think about:

```js
const defaultCreateCacheKey = (action, name) => {
  const { type, basename, location } = action
  const { pathname, search } = location
  return `${name}|${type}|${basename}|${pathname}|${search}` 
}
```

You'll notice we don't cache using the URL hash, as in most apps, it has the same data needs as the rest of the URL. As an example, here's how you could customize that though: 

```js
options.createCacheKey = (action, name) => {
  const { namespace, type, hash, basename, location } = action
  const { pathname, search } = location
  return `${name}|${namespace}|${type}|${basename}|${pathname}|${search}|hash` 
}
```


The cache key you use goes hand in hand with the ways you may want to clear it. To clear it, you can call: 

```js
route.onCallback = (request) => {
  const invalidator = 'myNamespace'
  request.cache.clear(invalidator)
}
```

In this case, all routes, and all the possible URLs they may have, and all callbacks for those URLs will be cleared in the namespace, `myNamespace`. If you wanted to clear all URLs for a single route/type, you would do this:

```js
route.onCallback = (request) => {
  const invalidator = 'MY_ROUTE'
  request.cache.clear(invalidator)
}
```

If for whatever reason, you wanted less info in your cache keys, you could do this:

```js
options.createCacheKey = (action, name) => {
  return `${action.location.pathname}` 
}
```

And that would require you to know the exact pathname to clear. 

As you can see you have lots of options...You can also dispatch an action to clear the cache, see [docs/action-creators.md](./action-creators.md) for more info on that. The only reason you'd want to do that--triggering your reducers to do more work--is if you want to listen to this action for reasons of your own.

If you'd like to disable caching for a route, you can set it:

```js
route.cache = false
```

or inversely disable it for an entire module, but enable it on a per route basis:

```js
options.cache = false
route.cache = true
```



## `shouldTransition(action, api): boolean`

Modify this to customize whether the Respond async middleware should snatch an action out of the synchronous store pipeline. Here's the default implementation:

```js
import { PREFIX } from 'respond-framework/types' //  '@@respond'

options.shouldTransition = (action, { routes }) => {
  const { type = '' } = action
  const route = routes[type]
  return route || type.indexOf(PREFIX) > -1
}
```


## `compose(middlewares, api, killOnRedirect = false) => (request): Promise<any>`

`compose` should rarely be customized. **Customize at your own risk.** 

`compose` is the backbone of Respond. It's essentially the route transition runner. It's modeled after [koa compose](https://github.com/koajs/compose), but with lots of customizations for our routing use case. Such customizations revolve around redirects, short-circuiting (returning `false`) and special handling of the browser back/next buttons. It can get a bit complicated, but for the brave, here's how you customize:

```js
import { compose } from 'respond-framework/core'
import trackMixpanel from './middleware/track-mixpanel'

export default (middlewares, api, killOnRedirect) => {
  return compose([...middlewares, trackMixpanel], api, killOnRedirect)
}
```

Imagine you want to provide a custom `compose` to your team to use in modules, and guarantee that some middleware is applied everywhere, that's one way to do it.

If you want to completely customize `compose`, dig up the source and get to work. It's not recommended, but genius never is :)

> The core is available like so in order to minimize friction related to forking. As easy as forking is, the additional hurdle often becomes the reason you never discover your genius solution. Contrary to popular approaches, we believe this is the correct philosophy for serious apps that are under *no illusion that custom apps won't call for custom solutions*. 

> Feel free to explore new ways Respond can be molded. Respond is all about scaling large codebases of serious custom apps.



## `formatRoute(inputRoute, type, routes): route`
If provided, this function is called for every route on creation. Use it to generate callbacks and add anything useful to your route objects. Here's an example:

```js
options.formatRoute = (route, type, routes) => {
  if (route.alert) {
    route.onEnter = () => alert(route.alert)
  }

  return route
}
```

Also note, if you provide a function for your route, ie:

`MY_ROUTE: ({ api }) => api.fetch('something')`

it will automatically be assigned to `route.thunk`. This is a built-in feature. If, for example, you customize your middleware and don't have the `pathlessRoute('thunk')` middleware, you can use this as an opportunity to set different default handling for functions.

Lastly note: `MY_ROUTE: '/path'` is already transformed into `{ path: '/path' }` before being passed to your handler.


## `onError(request, action)`

Respond runs with a default `onError` handler that will be called on any errors, regardless of what middleware you're using. It's called by the same code that powers the `call` middleware, but it's *not a middleware.* 

The default handler simply dispatches an error action. So if you're route is `MY_ROUTE`, `MY_ROUTE_ERROR` will be dispatched. 

The `request` object passed to `onError` will contain the error thrown on the `error` key and its specific type on `errorType`. The `location` state will contain the same key/vals as well once the action hits the reducer.

If you override this method, you can choose to not dispatch anything or dispatch something different, while perhaps doing some logging with a 3rd party service. 

Lastly, you can disable this feature altogether like so:

```js
options.onError = null
```




## Additional Info:

- `api` argument shape:

```js
{ 
  routes,
  history,
  options,
  ctx
}
```

> `ctx` contains state that can be stored across requests.


- `request` shape is documented [here](./request-shape.md)





