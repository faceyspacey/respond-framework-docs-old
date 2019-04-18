## Caching

Respond caches route callbacks. The data itself isn't in fact "cached." Rather, it's assumed the data dispatched from callbacks is in the store. 

Therefore the cache serves the purpose of skipping slow route callbacks in the middleware pipelines.

## Default Behavior

The default behavior is for thunks to be cached:

```js
const options = {
  middlewares: [
    // ...
    call('thunk', { cache: true }),
    // ...
  ],
```

*using this cache key:*

```js
  createCacheKey: (action, name = 'thunk') => {
    const { type, basename, pathname, search } = action
    return `${name}|${type}|${basename}|${pathname}|${search}`
  }
}

export default createModule(config, options)
```

## Example

Take this route:

```js
MY_TRADES: {
  path: '/my-grades',
  thunk: ({ api }) => api.get('trades')
}
```

An imagine your app has different languages, and currently the english `basename` is in use--here's what the cached route would look like:

```js
const cachedRoutes = {
  'thunk|MY_TRADES|en|my-grades|': true
}
```

The URL visited was: `/en/my-trades`. No search was used. The store had: action type `MY_TRADES.complete` dispatched.

 And the `trades` reducer now contains an array of trades. The `tradesById` reducer contains the trades hashed by ID.

```js
trades = [trade, trade, etc]
tradesById = {
  'iusfsdsfshtqwg': trade
  'qwoimndsuiyret': trade,
  'reuyiwkjvdfsdd': etc
}
```

The next time you visit `/en/my-trades`, `routes.MY_TRADES.thunk` will not be called, and `state.location.ready` will be `true` as soon as `MY_TRADES` is dispatched. In other words, the transition to this route will be instant. No possibly 3 second+ delay while the user waits for data on slow connections. Instead it's 0ms. 


## Cache Clearing

[Cache clearing](./action-creators.md#cacheClearing) is covered in depth in the action creators doc. But let's give you one example here if you're not in the mood for the exhaustive breakdown just linked:

```js
SUBMIT_TRADE: async ({ api, actions, clearCache, types }, { payload }) => {
  const { error } = await api.post('trade', payload)

  if (error) return actions.flashMessage('Your trade failed!', error)

  clearCache(types.MY_TRADES)
  return actions.myTrades()
}
```

Now, if you're looking at that and thinking "damn I have to clear the cache every time a user submits or edits data now!?" The answer is NO. *Don't forget what you already know about Redux.* The `trades` and `tradesById` reducer will already be updated by virtue of the `SUBMIT_TRADE` action:

```js
const trades = (state = [], action, types) => {
  switch(action.type) {
    case types.SUBMIT_TRADE:
      return [action.payload, ...state]
    case types.MY_TRADES_COMPLETE:
      return action.payload
    default:
      return state
  }
}
```

**So why did you call `actions.clearCache(types.MY_TRADES)`?**

To get you thinking. It's a contrived example. In a good system of this kind, maintaining a fresh cache is supposed to be highly automated. Calling `clearCache` is an *escape hatch*. A real example where you might use it is when other users of your app might have updated relevant data, and you need to be sure you have the latest. You also could simply turn off automatic caching like so:

```js
TRADES: {
  path: '/trades',
  thunk: ({ api }) => api.get('trades/all'),
  cache: false
}
```

So the idea here is that before you were in the private area of a cryptocurrency exchange, where a user can manage their past trades; and *now* you're on a main public page, where you want up to the moment latest trades pulled from your API. 

Presumably, you have some sort of caching deeper in your stack on the server side, e.g. memcache, so the data isn't unnecessarily fresh that it eats up your servers. But it's also a lot fresher than the 30 minutes your users spend navigating around your app. 

Either way, it's up to the server; from the perspective of the client it's never cached, and always hungry for that *new new*.



## Prefetching

This setup provides some very nice prefetching opportunities. It means you can prefetch any action, and have the data waiting for you, without actually navigating the user to that route. Respond provides automated prefetching via the `next` property of your routes:

```js
LOGIN: '/login',
TRADES: // same as above
MY_TRADES: // same as above
TRADE: {
  path: '/trade/:id',
  thunk: ({ api, state }, { params }) => {
    if (state.tradesById[params.id]) return
    return api.get(`trade/${id}`)
  }
},
HOME: {
  path: '/',
  next: {
    TRADES: true,
    MY_TRADES: {
      cond: request => !!request.state.user, // make sure the user is logged in
      fallback: 'LOGIN'
    },
    TRADE: {
      cond: request => !!request.state.user,
      targets: request => request.state.trades.map(trade => ({
        type: 'TRADE',
        params: { id: trade.id }
      })),
      fallback: 'LOGIN'
    }
  }
}
```

This allows the homepage to prefetch:

- `'/trades'`
- `'/my-trades'`
- and the trade detail pages (e.g. `'/trade/qwoimndsuiyret'`) for all the trades on `MY_TRADES`.

If the conditions (`cond`) are not met, they will not run. Rather, the fallback, `LOGIN` will be run. If a fallback appears multiple times, it will only be run once. 

You can provide a `targets` value for dynamic routes that could render more than one page (based on params, query, hash, etc).

So in the above case, potentially 12 routes would be prefetched just by virtue of being on the homepage. That gives a whole new meaning to the word ***"Next"*** in the React community!!

Also note: you don't need to manually prefetch in component code like you do in *NextJS*:

```js
<Link prefetch path='my-trades' />
```

That's not necessary.


> Along with prefetching data, corresponding webpack chunks are also prefetched. Double wami! If you want to just prefetch the chunk, you could do for example: `TRADES: 'chunk'`. That way you don't bother prefetching data you always want fresh anyway.



## StateCharts + Automated Pathfinder-based Testing

In the future, we will be able to generate both StateCharts and test sequences for *all possible routes users could take* much like [xstate](https://github.com/davidkpiano/xstate) is working on.

In other words, the `next` feature set collapses 4 very important capabilities:

- automated data prefetching
- automated webpack chunk prefetching
- StateChart generation
- generation of fully automated tests (every possible navigation sequence via a straightforward *"pathfinder"* algorithm)


Yea, another **world first** for team *First Responders*. To anyone who'se been tracking the frontend development space for some time, this is HUGE. Hurray!


## `action.prefetch`

If you're deciding to use callback caching, you need to make sure your `ROUTE.complete` callbacks don't trigger UI state in any way that is undesirable. For example, if you had:

```js
const sidebarOpenReducer = (state = false, action, types) => {
  switch(action.type) {
    case types.MY_TRADES.complete:
      return true
    // etc
    default:
      return state
  }
}
```

Then the sidebar would unexpectedly close while prefetching a route. If you really needed such behavior, your reducers must check the `prefetch` flag on the action:

```js
const sidebarOpenReducer = (state = false, action, types) => {
  if (action.prefetch) return state

  switch(action.type) {
    case types.MY_TRADES.complete:
      return close
    // etc
    default:
      return state
  }
}
```


## Further Reading

- creating custom cache keys: [options.createCacheKey](./advanced-options.md#createcachekey-action-name-string)
- cache clearing in depth: [actions.clearCache](./action-creators.md#cacheClearing)