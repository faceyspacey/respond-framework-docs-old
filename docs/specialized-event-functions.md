# Specialized Event Functions

Most of the time event functions ("action creators" in Redux terminology) generated from your events map should suffice, but there are occassions where more specialized event functions can be of assistance.

All specialized event functions can be imported from `respond-framework/events`:

```js
import {
  push, 
  replace,
  back,
  next,
  jump,
  reset,
  set,
  setParams,
  setQuery,
  setState,
  setHash,
  setBasename,
  redirect,
  notFound,
  confirm,
  clearCache,
  changeBasename,
} from 'respond-framework/events'
```


## Basic History Event Functions

Typically you shouldn't dispatch path strings, but the event functions `push` and `replace` fit the React Native use case where a dynamic deep link needs to be transformed into an event object.

`push` and `replace` should be used *only* when you have a URL to work with and want to translate it into an event, such as when receiving deep links or push notifications on React Native.

| name | arguments | 
| --- | --- | 
| `push` | `path: string`, `state?: Object` | 
| `replace` | `path: string`, `state?: Object` | 


The reason the correct way to change routes is with events is because that allows you to easily change the URLs your users see without changing much application code.

> There may be times where it makes sense in the browser, but you better have a good reason (for example, the user literally enters the URL they would like to go to).


## Back/Next Event Functions

These event functions serve a specific purpose: when you don't want to think about what possible route the user came from to reach the current route, but you want to reliably send them backward or perhaps forward without having to think of what event to create.

You can optionally pass a second argument for `state` you want attached to the history entry.

| name | arguments | 
| --- | --- | 
| `back` | `state?: Object` | 
| `next` | `state?: Object` | 

They work reliably because Respond perfectly mirrors the history entries stored in state to the browser's hidden one. This solves many problems, one of which being the potential of bouncing a visitor from your site. `back` and `next` (as well as `jump` below) have a required `fallback` option which can be used to trigger another event instead:

```js
const BackButton = (props, _, { back, step1 }) =>
  Button({
    onClick: back({ fallback: step1() }),
  }, 'BACK')
```

> Respond is the new frontier when it comes to accurate `history` API usage. Check the [History Mirroring](./history-mirroring.md) doc for a deep dive on this subject.


## Usage with the `Link` component


You can use `back`, `next` and *all specialized event functions* in the `Link` component as well:

```js
import Link from 'respond-framework/Link'

const BackLink = (props, _, { links: { back, step1 } }) =>
  Link({
    to: back({ fallback: step1() }),
  }, 'BACK')
```

Here, we use `events.links` to access event functions that immediately return event objects ("actions" in Redux terminology) instead of automatically dispatching functions wrapped in `useCallback`.


## Advanced History Event Functions

The following advanced history event functions offer capabilities that have never before been available on the web.

Because we are able to rewrite the entire history entries stack that the browser hides from us, we can in fact reset it to whatever you want, or jump far away.

| name | arguments | 
| --- | --- | 
| `jump` | `delta: number \| string`, `byIndex?: boolean`, `direction?: 1 \| -1` | 
| `reset` | `entries: Array<path \| Event \| [path, state, ?key]>`, `index?: number`, `direction?: 1 \| -1` | 


`jump` receives as its first argument one of:

- a string where it's the unique key of one of the history `entries`
- a positive or negative number where it's the distance you want to jump to another entry *(default)*
- a positive number starting from 0 where it's the index of a precise entry you wan to jump if the 2nd argument `byIndex` is true


`reset` receives an array of event objects, path strings, or arrays each containing a path, state object and optional key string. Given whatever you pass it, events will be generated, and URLs instantly replayed on the browser without the user being able to perceive they were added to the history stack. The `index` is where in the new stack you'd like to drop the user. 

For both event functions, `direction` relates to the direction you want to simulate the user as coming from, whose state can be used in components for horizontally animated transitions, such as in Stack Navigators.

### Here's a scenario where `reset` may come in handy:

- the user may have just arrived on your site and has only visited a single URL--perhaps the user arrived on step 3 of your checkout funnel
- you want to generate the first 2 steps, so the user can press *back* to them
- you can dispatch `reset([step1Event, step2Event, step3Event])` and not only will the redux state match, but if the user presses back, the history shifts to those URLs that the user hasn't even visit yet

> `route.defaultEntries` is the preferred way to generate the above `reset` sequence on initial load:

```js
step3: {
  path: '/checkout/step-3',
  defaultEntries: request => [step1Event, step2Event, step3Event]],
}
```

## `set` Event Functions

`set` event functions have a special characteristic: they skip right to the reducer, skipping the middleware pipeline. Of course they also change the browser history as well. 

| name | arguments | 
| --- | --- |
| set | `event: Event \| Function`, `delta?: number \| string`, `byIndex?: boolean` | 
| setParams | `params: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` | 
| setQuery | `query: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` |
| setState | `state: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` |
| setHash | `hash: string \| Function`, `delta?: number \| string`, `byIndex?: boolean` | 
| setBasename | `basename: string` \| Function, `delta?: number` \| `string, byIndex?: boolean` |

**usage:**

```js
set({ params: { slug: 'thunk-life' }})
```

This will change the params for the current history stack entry, in other words the current URL. 

If you provide the 2nd and possibly 3rd arguments you can make changes on different entries in the stack. They operate the same way as in the `jump` above:

```js
set({ params: { slug: 'thunk-life' }}, 3, true)
```

Calls to the others, just call `set` behind the scenes by generating an event object containing just, for example, `params`:

```js
setParams({ slug: 'thunk-life' })
setQuery({ keywords: 'redux 2020' })
setState({ flashMessage: 'you entered the wrong password' })
setHash('foo')
setBasename('es')
```

You can also provide a function as the first argument if you want to interact with the current value:

```js
setState(state => ({ num: state.num + 1 }))
```

The result will be merged into the current state (which is the same behavior as if you provided an object). To remove a value, set it to `undefined`. 


## Miscellaneous Event Functions

| name | arguments | 
| --- | --- | 
| `changeBasename` | `basename: string`, `event?: Event` |
| `redirect` | `event: Event`, `status?: Number = 302` |
| `notFound` | `state?: State` | 
| `confirm` | `canLeave?: boolean = true` | 
| `clearCache` | `invalidator?: string \| function \| Event`, `callbackNames?: Array<string>` |


- **`changeBasename`**
  - `basename` (string, required)
  - `event` (Event, optional)

  Pass the name of the basename you want to change to. It must be one of the basenames provided upfront in `options.basenames[]`. Optionally pass an event object to change routes as you change basenames. 

- **`redirect`**
  - `event` (Event, required)
  - `status` (Int, optional, default: 302)

  Generally you don't need to use `redirect` directly. Returned events in callbacks are automatically tranformed into redirects. Use this directly in handlers attached to UI elements or in pathless events. The purpose is to replace the current history entry. 

  ```js
  import { redirect } from 'respond-framework/events'

  Button({ onClick: redirect(events.foo()) })
  ```
  *as a pathless event:*

  ```js
  pathlessEvent: {
    thunk: ({ state, events }) => !state.ok && redirect(events.bar())
  }

  const MyComponent = (props, _, events) =>  Button({ onClick: events.pathlessEvent() })
  ```

  ***not necessary in pathful events:***

  ```js
  pathfullEvent: {
    beforEnter: ({ state, events }) => !state.ok && redirect(events.bar()), 
    path: '/pathfull-event',
  }

  const MyComponent = (props, state, events) =>  Button({ onClick: events.foo() })
  ```

  ***instead the following suffices:***

  ```js
  pathfulEvent: {
    beforEnter: ({ state, events }) => !state.ok && events.bar(), // redirect automatically found
    path: '/pathfull-event', // because callbacks already knows you're in a route you would like to leave
  }
  ```

  *status codes:*

  However, an exception is if you would like to set a status code, such as 301s for permanent redirects in SSR:


  ```js
  foo: {
    beforEnter: ({ state, events }) => !state.ok && redirect(events.bar(), 301), 
    path: '/foo',
  }
  ```
  
- **`notFound`**
  - `state` (Object, optional)

  Dispatch the `notFound` event, and optionally attach some entry `state` to the URL. This is useful when something went wrong and you're not sure where to send the user, but at least want to provide an indicator.

  ```js
  posts: {
    path: '/posts',
    thunk: async ({ params, api }) => {
      const { posts } = await api.get('post/list', params)
      if (posts.length === 0) return notFound()
      return { posts }
    }
  }

  ```
  The built-in `notFound` event is not implemented as a "*special case*" and you can customize its options (such as the `path`) like any other event:

  ```js
  notFound: {
    path: '/fail-whale', // default: '/not-found'
    onEnter: ({ mixpanel, state }) => mixpanel.track('lost', state)
  }
  ```

  > Respond will also dispatch this event if anything goes wrong internally.

  The `notFound` event has special behavior: if *no* `notFound` event exists in the current module, one from the closest parent module will be dispatched, bubbling up to the built-in one attached to the top level events map. 

- **`confirm`**
  - `canLeave` (boolean, optional, default: true)

  Confirm the user can leave the currently blocked route. The user will have been blocked by any callback that returns `false`, usually `beforeLeave` or `beforeEnter`:

  ```js
  checkout: {
    path: '/checkout',
    beforeLeave: ({ state, events }) => {
      if (!state.didFinish) {
        events.displayModal('You have things in your cart. Are you sure you wanna leave?')
        return false
      }
    }
  }
  ```

  Once the user is blocked from leaving a route, calling `confirm` will transition the user to where they previously tried to go; and passing `false` to confirm will keep them where they are:

  ```js
  const ConfirmModal = (props, state, events) =>
    Modal({
      onConfig: events.confirm(), // OK button
      onCancel: events.confirm(false), // CANCEL button
      messag: state.confirmText,
    })
  }
  ```

- **`clearCache`**
  - `invalidator` (string | Event | function, required)
  - `callbackNames` (Array<string>, optional)

  The Respond store is one big cache for your data. Therefore callbacks in your events map don't need to cache any data themselves, but just record what URLs and what callbacks have been accessed via a cache key:

  ```js
  `${callbackName}|${eventName}|${basename}|${pathname}|${search}`
  ```

  By default Respond caches all thunks so they don't need to be called a second time if the same URL is visited. Your cache might look like:
  
  ```js
  const cache = {
    'thunk|home|en|/home|?referrer=google': true,
    'thunk|login|en|/login|': true,
    'thunk|dashboard|en|/dashboard|': true,
  }
  ```
  Cached callbacks won't run if it's marked as cached. If you think it's stale, you can clear it any time with the `clearCache` event function. 
  
  ```js
  clearCache('dashboard') // clears any cache item containing this string
  ```

  You can easily clear multiple items in one go:
  
  ```js
  checkoutComplete: {
    path: '/thank-you',
    onComplete: () => clearCache('thunk') // clear all cached thunks
  }
  ```
  
  The most common way to clear cache entries is to provide a string. All cache keys will be searched for an occurrence of this string. This allows you to remove many items with a short string as above or a single item with a long specific string. You can also provide an `Event` object which will be converted to a complete cache key capable of deleting a single item. 
  
  ```js
  cart: '/cart',
  checkout: {
    path: '/checkout',
    onComplete: ({ events }) => clearCache(events.cart()) // single URL cleared
  }
  ```

  Lastly, you can provide a function which will be passed the entire cache hash and the Respond `api`. It is up to you exactly how and what items are removed from the cache. The function can either mutably delete items in the hash
  
  ```js
  const myInvalidator = (cache, api) => {
    for (const k in cache) {
      if (k.indexOf('thunk|someEvent')) delete cache[k]
    }
  }

  clearCache(myInvalidator)
  ```

  or return a new one:

  ```js
  const myInvalidator = (cache, api) =>
    Object.keys(cache
      .filter(k => e.indexOf('someEvent') === -1)
      .reduce((newCache, k) => ({ ...newCache, [k]: true }), {})

  clearCache(myInvalidator)
  ```

  If no `invalidator` is provided, the entire cache is cleared.
  
  ```js
  clearCache()
  ```

  Optionally `callbackNames` can be passed as a 2nd argument to `clearCache`, in order to insure only callbacks with the names provided are cleared. This only applies when passing an event object as the 1st argument. It's unnecessary unless you're caching more than one kind of callback, as the default Respond middleware only caches `thunks`:

  ```js
  clearCache(eventObject, ['thunk', 'onComplete'])
  ```

  > See [options.createCacheKey](./advanced-customization-options.md#create-cache-key) for more info on how to customize the cache key created. 






