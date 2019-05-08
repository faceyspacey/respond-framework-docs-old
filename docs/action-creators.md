# Action Creators

Having learned from [redux-first-router](https://github.com/faceyspacey/redux-first-router), all action creators below have been carefully selected. There are no legacy action creators. There are specific use cases for each action creator.

For example, typically you shouldn't dispatch path strings (as that couples your code to URLs which could change), but action creators `push` and `replace` fit the React Native use case where a dynamic deep link needs to be transformed into an action.


```js
import {
  redirect,
  notFound,
  confirm,
  clearCache,
  changeBasename,
  back,
  next,
  push, 
  replace,
  jump,
  reset,
  set,
  setParams,
  setQuery,
  setState,
  setHash,
  setBasename
} from 'respond-framework/actions'

```

## Main Action Creators

| name | arguments | 
| --- | --- | 
| `changeBasename` | `basename: string`, `action?: Object` |
| `redirect` | `action: Object`, `status?: int = 302` |
| `notFound` | `state?: Object` | 
| `confirm` | `canLeave?: boolean = true` | 
| `clearCache` | `invalidator?: string \| function \| object`, `callbackNames?: Array<string>` |


- **`changeBasename`**
  - `basename` (string, required)
  - `action` (Object, optional)

  Pass the name of the basename you want to change to. It must be one of the basenames provided upfront in `options.basenames[]`. Optionally pass an action to change routes as you change basenames. 

- **`redirect`**
  - `action` (Object, required)
  - `status` (Int, optional, default: 302)

  You often don't need this action creator. Actions from one route to another in callbacks are automatically tranformed into redirects. Use this directly in handlers attached to UI elements or in pathless routes. The purpose is to replace the current history entry. 

  ```js
  import { redirect } from 'respond-framework/actions'

  <Button onClick={actions.foo.redirect({ param: props.param })} />
  ```
  *as a pathless route:*

  ```js
  PATHLESS_ROUTE: {
    thunk: ({ state }) => !state.ok && actions.foo.redirect()
  }

  const MyComponent = (props, state, actions) =>  <Button onClick={actions.pathlessRoute} />
  ```

  *not necessary:*

  ```js
  ROUTE: {
    thunk: ({ state }) => !state.ok && actions.foo(), // redirect automatically discovered
    path: '/route', // because Respond already knows you're in a route you would like to leave
  }

  const MyComponent = (props, state, actions) =>  <Button onClick={actions.route} />
  ```


  It's also useful to set the precise status code during SSR, commonly changing it to 301 for permanent redirects.
  
- **`notFound`**
  - `state` (Object, optional)

  Send the user to the `NOT_FOUND` route, and optionally attach some entry `state` to the URL. This is useful when something went wrong and you're not sure where to send the user, but at least want to provide an indicator. You can customize what happens on the `NOT_FOUND` route like any other route:

  ```js
  NOT_FOUND: {
    path: '/fail-whale', // default: '/not-found'
    onEnter: ({ mixpanel, state }) => mixpanel.track('lost', state)
  }
  ```

  > Respond will also dispatch this route if anything goes wrong internally.

  `NOT_FOUND` has special behavior: if no `NOT_FOUND` route exists in the current module, the `NOT_FOUND` route of the closest parent module will be dispatched, falling back to the built-in one. 

- **`confirm`**
  - `canLeave` (boolean, optional, default: true)

  Confirm the user can leave the currently blocked route. The user will have been blocked by any callback that returns `false`, usually `beforeLeave` or `beforeEnter`

  ```js
  ROUTE: {
    path: '/foo',
    beforeLeave: ({ state, dispatch, actions }) => {
      if (!state.didFinish) {
        actions.displayModal('Are you sure you wanna leave?')
        return false
      }
    }
  }

  const ConfirmModal = (props, state, actions) => {
    return (
      <Modal
        onConfirm={actions.confirm} // OK button
        onCancel={() => actions.confirm(false)} // CANCEL button
        message={state.confirmText}
      />
    ) 
  }
  ```

  > Call `confirm` if the user presses OK and *confirms* they want to go where they initially intended. Pass `false` to leave them where they are.

- **`clearCache`**
  - `invalidator` (string | Action | function, required)
  - `callbackNames` (Array<string>, optional)

  Redux--ahem, *Remixx*--is one big cache for your data. Therefore your routing callbacks don't need to cache any data themselves, but just record what URLs and what callbacks have been accessed. Here's what the default hash of cached items look like:
  
  ```js
  const cache = {
    // `${name}|${type}|${basename}|${pathname}|${search}`: true, // code for default cache key
    'thunk|HOME|en|/en/home|?referrer=google': true // concrete example
  }
  ```
  Cached callbacks like thunks won't run if it's marked as cached. If you think it's stale, you can clear it any time with this action creator. You can easily clear multiple items in one go:
  
  ```js
  FINISH: {
    path: '/finish',
    onComplete: () => clearCache('thunk') // clear all cached thunks
  }
  ```
  
  The most common way to clear cache entries is to provide a string. All cache keys will be searched for an occurrence of this string. This allows you to remove many items with a short string as above or a single item with a long specific string. You can also provide an `Action` object which will be converted to a complete cache key capable of deleting a single item. 
  
  ```js
  CART: '/cart',
  CHECKOUT: {
    path: '/checkout',
    onComplete: ({ actions }) => clearCache(actions.cart()) // single URL cleared
  }
  ```

  Lastly, you can provide a function which will be passed the entire cache hash and the Respond `api`. It is up to you exactly how and what items are removed from the cache. The function can either mutably delete items in the hash, or return a new one:
  
  ```js
  const myInvalidator = (cache, api) => {
    for (const k in cache) {
      if (k.indexOf('thunk|ROUTE')) delete cache[k]
    }
  }
  ```

  If no `invalidator` is provided, the entire cache is cleared. Optionally `callbackNames` can be passed as a 2nd argument to `clearCache`, in order to insure only callbacks with the names provided are cleared. This only applies when passing an action object as the 1st argument, and is irrelevant unless you're caching more than one kind of callback. The default Respond middleware only caches `thunks`:
  
  ```js
  const defaultMiddlewares = [
    // others
    call('thunk', { cache: true }),
    // others
  ]
  ```

  See [options.createCacheKey](./advanced-customization-options.md#create-cache-key) for more info on how to customize the cache key created, and accordingly what sort of strings you must provide to match items you want to invalidate. 



## History Action Creators

The `push` and `replace` action creators should be used *only* when you have a URL to work with and want to translate it into an action, such as when receiving deep links or push notifications on React Native. 

There may be times where it makes sense in the browser, but you better have a good reason (for example, the user literally enters the URL they would like to go to).

| name | arguments | 
| --- | --- | 
| `push` | `path: string`, `state?: Object` | 
| `replace` | `path: string`, `state?: Object` | 


The reason the correct way to change routes is with actions is because that allows you to easily change the URLs your users see without changing much application code.



## Back/Next Action Creators

These action creators serve a specific purpose: when you don't want to think about what possible route the user came from to reach the current route, but you want to reliably send them backward or perhaps forward without having to think of what action to form.

You can optionally pass a second argument for `state` you want attached to the history entry.

| name | arguments | 
| --- | --- | 
| `back` | `state?: Object` | 
| `next` | `state?: Object` | 

They work reliably because Respond perfectly mirrors the history entries stored in state to the browser's hidden one. This solves many problems, one of which being the potential of bouncing a visitor from your site. `back` and `next` (as well as `jump` below) safe-guarded from that by throwing an exception. Which is an indicator you should use Respond's `canJump` selector:

```js
const MyComponent = (props, { location }, { back, step1 }) => (
  <button onClick={location.canJump(-1) ? back() : step1()}>
    back
  </button>
)
```

> Respond is the new frontier when it comes to accurate `history` API usage. Check the [History Mirroring](./history-mirroring.md) doc for a deep dive on this subject.


## Advanced History Action Creators

These are some bad-ass action creators that have never before been available on the web. Because we are able to rewrite the entire history entries stack that the browser hides from us, we can in fact reset it to whatever you want, or jump far away. Here's an example of `reset`:

- the user may have just arrived on your site and has only visited a single URL--perhaps the user arrived on step 3 of your checkout funnel
- you want to generate the first 2 steps, so the user can press *back* to them
- well, you can dispatch `reset([step1action, step2action, step3action])` and not only will the redux state match, but if the user press back, the URL change to those URLs that the user didn't even visit yet:

| name | arguments | 
| --- | --- | 
| `jump` | `delta: number \| string`, `byIndex?: boolean`, `direction?: 1 \| -1` | 
| `reset` | `entries: Array<path \| Action \| [path, state, ?key]>`, `index?: number`, `direction?: 1 \| -1` | 


`jump` receives as its first argument that can be one of:

- a string where it's the unique key of one of the history `entries`
- a positive or negative number where it's the distance you want to jump to another entry *(default)*
- a positive number starting from 0 where it's the index of a precise entry you wan to jump if the 2nd argument `byIndex` is true


`reset` receives an array of action objects, path strings, or arrays each containing a path, state object and optional key string. Given whatever you pass it, actions will be generated, and URLs instantly replayed on the browser without the user being able to perceive they were added to the history stack. The `index` is where in the new stack you'd like to drop the user. 

For both action creators `direction` relates to the direction you want to simulate the user as coming from.



## Set Action Creators

`set` action creators have a special characteristic: they skip right to the reducer, skipping the middleware pipeline. Of course they also change the browser history as well. 

| name | arguments | 
| --- | --- |
| set | `action: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` | 
| setParams | `params: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` | 
| setQuery | `query: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` |
| setState | `state: Object \| Function`, `delta?: number \| string`, `byIndex?: boolean` |
| setHash | `hash: string \| Function`, `delta?: number \| string`, `byIndex?: boolean` | 
| setBasename | `basename: string` \| Function, `delta?: number` \| `string, byIndex?: boolean` |

Calls to the action creators other than `set`, just call `set` behind the scenes by generating an action containing just, for example, `params`. 

Here's an example:

```js
set({ params: { slug: 'thunk-life' }})
```

This will change the params for the current history stack entry, in other words the current URL. If you provide the 2nd and possibly 3rd arguments you can make changes on different entries in the stack. They operate the same way as in the `jump` action creator above.

You can also provide a function as the first argument if you want to interact with the current value:

```js
setState(state => ({ num: state.num + 1 }))
```

The result will be merged into the current state (which is the same behavior as if you provided an object). To remove a value, set it to `undefined`. 