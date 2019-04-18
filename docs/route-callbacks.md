# Route Callbacks

Respond comes with a default middleware, `call`, which is responsible for calling the callbacks attached to your routes. It's used in multiple places (6 to be exact). 

Chances are--even if you use custom middleware for sagas, observables or GraphQL--you will use it at least once in your custom middleware pipeline.

> Refer to the [call middleware](./middleware.md#call) docs to learn more how to customize it.



## Overview

The 6 callbacks as called by the default middleware is as follows:

```js
middlewares: [
  // short-circuiting middleware:
  pathlessRoute('thunk'),
  serverRedirect, 
  anonymousThunk,

  // pipeline starts here:
  transformAction,
  call('beforeLeave', { prev: true }),
  call('beforeEnter'),
  enter, // STATE CHANGES: original action dispatched to store
  changePageTitle,
  call('onLeave', { prev: true }),
  call('onEnter'),
  call('thunk', { cache: true }),
  call('onComplete')
]
```

**Example Usage:**

```js
MY_CRYPTO_TRADES: {
  path: '/my-trades',
  beforeEnter: ({ state }) => !!state.user, // 
  thunk: ({ api }) => api.get('trades'),
  onComplete: () => window.scrollTo(0, 0) // scroll to top after trades are loaded
}
```




The important thing to observe is the placement of the `enter` middleware, which appears after `beforeLeave` and `beforeEnter`. The Respond middleware pipeline captures the inital action and holds on to it until it until the `enter` middleware is reached. You can bail out at any time, deciding not to change routes.

Unlike the original Redux middleware which is synchronous, async work can occur throughout the entire pipeline and within all its callbacks. So that means you can even check a flag in the database, `await` it and then decide what to do.

Some things you can do:

- **return `false`** to short-circuit the pipeline
- dispatch a **"follow-up action"** containing data from the server
- **automatically dispatch by returning an action**
- **redirect** to another route by dispatching its corresponding action
- perform **DOM-related work**, such as scroll restoration after the next state is entered
- address a laundry list of cross-cutting concerns where precise **linear coordination** is important


## Options Level Callbacks

Callbacks can also exist on the options for a given module:

```js
const options = {
  beforeLeave: saveScroll,
  onEnter: restoreScroll
}

export default createModule(config, options)
```

They will be called in parallel to route level callbacks of the same name. You can also configure it so options level callbacks are only used as a fallback in the case that one doesn't exist on the route. See the [options](./options.md) doc for how to configure that.


Now let's take a deep dive into each callback:



## `beforeLeave`

`beforeLeave` is called before you leave a given route. 

> It's called on the route you're leaving, not the route you're going to. 

As with all other callbacks, you can do anything here, but a common task is "blocking" by returning `false`:


```js
CHECKOUT: {
  path: '/checkout',
  beforeLeave: ({ state, dispatch, actions }) => {
    if (!state.didFinish) {
      actions.displayModal('Are you sure you wanna leave?')
      return false // Respond pipeline stops here
    }
  }
},
DISPLAY_MODAL: {}
```

Returning `false` in any callback immediately short-circuits the middleware, preventing it from calling subsequent middleware. That means if you `return false` before the initial action is dispatched to the store, the state will never change. 

Respond comes with an easy pattern to confirm whether users want to leave a given route:

```js
const ConfirmModal = (props, state, actions) => {
  if (!state.modalText) return null // modalText reducer contains 'Are you sure...'

  return (
    <Modal
      onConfirm={actions.confirm} // OK button
      onCancel={actions.cancel}   // CANCEL button
      message={state.modalText} 
    />
  ) 
}
```

Built-in actions, `confirm` and `cancel`, are aware of the route transition just blocked. If the user confirms, the route previously attempted will now be completed. If `canceled`, things will be set back to normal without leaving the route.

> NOTE: if you need plain jane actions, just create a route with an empty object as its value, such as `DISPLAY_MODAL`.

**Here are some other ideas of things you can do in the `onLeave` callback:**

- record scroll position, for perfect scroll restoration on return
- store other state on the history entry for retreival on return

## `beforeEnter`

`beforeEnter` is another callback often used for short-circuiting the pipeline. Imagine you're under 21, and you want to enter the site for your favorite beer, Chimay, right. You enter the year you were born, 1999:

```js
BEER: {
  path: '/hoegarden'
  beforeEnter: ({ state }) => state.age < moment().subtract(21, 'years')
}
```

**BAM!** You were stupid enough to not lie about your age and you don't get to learn about your favorite beer.

> Not fond of the example? I'm sure you can imagine a worse one.


What if we don't have the data we need to make the determination:


```js
TOP_SECRET: {
  path: '/shh'
  beforeEnter: async ({ api, state, actions }) => {
    const { allowed, reason } = await api.verify(state.user)

    if (!allowed) {
      actions.flashMessage(reason)
      return false
    }
  },
  thunk: ({ api }) => api.fetch('top-secret-data')
}
```

The observant among you might have wondered where `state.age` came from in the first example. It was contrived. Here's *one way* you might implement that which highlights the async nature of our pipline:

```js
routes: {
  BEER: {
    path: '/hoegarden'
    beforeEnter: async ({ state }) => {
      const age = await new Promise(resolve => {
        const message = 'How old are you?'
        const onSubmit = ({ age }) => resolve(age)
        actions.setState({ message, onSubmit })
      })

      return age < moment().subtract(21, 'years')
    }
  }
},
components: {
  BeerGarden: () => (
    <>
      <ConfirmAgeModal /> // overlayed on top of visible age until confirmed
      <Details />
    </>
  )
},


const ConfirmAgeModal = (props, { location }) => {
  const { message, onSubmit } = location // grab info right off history state for current entry
  return message && <FormModal onSubmit={onSubmit} message={message} />
}
```

Yea, that's pretty powerful. No longer must you glue a hodgepodge of action creator thunks, while losing mental context for how they all relate. It's all one sequential aysynchronous state change transition, *or in this case, one that's short-circuited.*


### I wish Redux had always been like this

Users coming from Redux have wanted these capabilities for a long time, *even if they didn't know it.* It would have been nice to have basic asynchronous capabilities without having to resolve to "nuclear options" like Sagas and observables long ago. It's nice to not have to dispatch multiple actions for one result. With thunks your usual workflow looked like this:

- 1st dispatch for a thunk function
- 2nd dispatch for what the thunk actually does
- and sometimes many more, or some composition of action creator functions

Here it's one seamless pattern. You'll be wondering why we haven't done this all along...And yes, it's possible for extremely reactive client-side apps to feel like Rails or a Rails-clone MVC. 


## `onLeave`

This callback is useful when you say goodbye to a route and are officially on the next route. *Any last words?* 

**Some things you may do are:**

- increment an important statistic with a 3rd party service
- leave a reminder message about something left behind


Keep in mind the callback is attached to a different route than what the current action corresponds to. Let's say a customer bails on a purchase and continues browsing:

```js
CONFIRM_PURCHASE: {
  path: '/checkout/step-3',
  onLeave: ({ action, types, actions }) => {
    if (action.type === types.PRODUCTS) {
      actions.flashMessage({
        message: 'You stil have 2 items in your cart. Would you like to finalize your purchase?',
        action: actions.confirmPurchase
      })
    }
  }
},
PRODUCTS: '/products'


const FlashComponent = (props, { flash }) => flash && (
  <div>
    <CloseButton onClick={flash.clear} />
    <p>{flash.message}</p>
    <BrandButton onClick={flash.action}>Complete Purchase</BrandButton>
  </div>
)
```


## `onEnter`

`onEnter` is called after `onLeave`, but is the first callback attached to the new route to be called. It's called after the state change is complete, and therefore the DOM is up to date. It's a place designated to setting up things that rely on the DOM being stable. Some examples:

- scroll restoration
- setting up mouse/scroll/resize/etc event handlers
- applying a non-react library, e.g. charts, carousel, etc

Let's checkout the example for scroll restoration hinted to at the top. This time let's use options level callbacks:

```js
const options = {
  beforeLeave: ({ location }) => {
    location.state.scrollPosition = [window.pageXOffset, window.pageYOffset]
  },
  onEnter: ({ location }) => {
    const [x, y ] = location.state.scrollPosition || [0, 0]
    window.scrollTo(x, y)
  }
}

export default createModule(config, options)
```

Yes, we know, you've been indoctrinated to never apply state directly, but it's just too easy, and guaranteed to compromise *no thing*. Be fearless, mighty developer. Rules are meant to be broken.

When you're ready to put on your suspenders, you can trade a bit of render perf for a grown-up state change:


```js
const options = {
  beforeLeave: ({ actions }) => {
    const scrollPosition = [window.pageXOffset, window.pageYOffset]
    actions.setState({ scrollPosition })
  },
  onEnter: ({ location }) => {
    const [x, y ] = location.state.scrollPosition || [0, 0]
    window.scrollTo(x, y)
  }
}
```


## `thunk`

Thunks play a very important role in Respond. The consensus among the Redux community has been that Sagas and Observables have been overkill in many of the places they've been used. With Respond, thunks have never felt so right. Let's take  alook at an example:

```js
BOOKS: {
  path: '/books/:category',
  thunk: ({ api }, { params }) => api.get(`items/${params.category}`)
}
```

If you've been reviewing the Respond docs, you've likely seen that pattern many times. This one liner shows the dominance of Respond over Redux "classic" workflows. *Long live Redux!* Development is supposed to be fun--so what do we get out of the above:

- no dispatching
- returns automatically dispatched
- state changes timed perfectly
- automatic caching based on the URL (fetching won't be performed on subsequent visits)
- automatically created action type
- automatically created action creator:`actions.list({ category: 'magick' })`
- easy access to everything you need like `params`
- less actions *(dual action pattern)*; obvious best practices where "fat/smarter reducers" listen to a fewer # of actions
- dependency-injected `api`

Of course you must setup `api` to be available, but with Respond this best practice is painfully obvious--and not to mention clearly documented:

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
    }
  }
})
```

Check the [dependency injection](./dependency-injection.md) doc for the full breakdown.

Aside from the modularity that Respond offers, its deepest beauty lies in the simplicity of the **"dual action pattern"** that brings us back to the Rails days. For any single route transition, you have at most 2 actions:

- the UI-triggered action
- the follow-up action which contains fetched data (the `thunk`)

```js
const middlewares = [
  enter, 
  call('thunk', { cache: true }),
]
```

Before Respond or [Redux-First-Router](https://github.com/faceyspacey/redux-first-router), your app was very likely to suffer from using actions as setters, and treating your store as mutable. Or you might be using lifecycle handlers to dispatch actions, and then SSR was a major problem for you. 

> *Or* your React codebase simply turned into a maze you could never find your way out of because your app lacked architecture and structure, and capabilities were stuffed who knows where in a mountain of components.

Respond makes it so your users hop from one *route-triggering action* to the *next*. Respond makes it so your *developer eyes* hop from one route to the next. And within one route, from *trigger action* to *follow-up action*. 

The flow is:

```js
const middlewares = [
  enter,                          // 1st trigger the state change
  call('thunk', { cache: true }), // 2nd the follow-up thunk returns the data
]
```

You can also reverse it like so:


```js
const middlewares = [ 
  call('thunk', { cache: true, start: true }),
  enter
]
```

In this setup, the URL doesn't change until you come back with the data. An initial action hits the store right when the user clicks, but you use that to show a loading indicator:

```js
components: {
  App: (props, { page, location }) => {
    const { ready, components } = location // Respond knows when your data is `ready`
    const Component = components[page]

    // insure the data is ready AND the component is loaded
    return ready && Component ? <Component /> : <Loading />
  }
}
```

By seeing both approaches, the elegance of the "dual action pattern" should be clear. 

### Reducers 

With this pattern, reducers will individually listen to a wider variety of actions as a result. 

> That's as opposed to pairing individual actions with individual reducers, resulting in an explosion in the count of both.

Picture a standard mobile app with a sidebar, bottom tab bar and a modal. Let's dissect how our reducers might handle that:

```js
const page = (state = 'home', action, types) => {
  switch(action.type) {
    case types.HOME:
      return 'home',
    case types.FEED:
      return 'feed'
    case types.PROFILE:
      return 'profile'
    case types.SETTINGS:
      return 'account'
    case types.ACCOUNT:
      return 'account'
    default:
      return state
  }
}

const sidebarOpen = (state = false, action, types) => {
  switch(action.type) {
    case types.HOME:
    case types.FEED:
    case types.PROFILE:
      return false,
    case types.SETTINGS:
    case types.ACCOUNT:
      return true
    default:
      return state
  }
}

const modalVisible = (state = false, action, types) => {
  switch(action.type) {
    case types.DISPLAY_MODAL:
      return true
    case types.FEED:
    case types.SETTINGS:
    case types.ACCOUNT:
      return false
    default:
      return state
  }
}

const bottomTabBarIndex = (state = 0, action, types) => {
  switch(action.type) {
    case types.HOME:
      return 0
    case types.FEED:
      return 1
    case types.PROFILE:
      return 2
    case types.SETTINGS:
    case types.ACCOUNT:
    default:
      return state
  }
}
```

Notice how there is no action to dispatch opening or closing the sidebar, showing or hiding the modal, changing the bottom tab bar index. **The reducers are smart enough to know what to do with a smaller number of routing actions.** *This is how your apps were always supposed to be developed, but most of you haven't. Shame shame shame.* 

Jk, we call this the "thin actions, fat reducers" approach. Or "less actions, smarter reducers." 

> For the new jacks among us, in the server-side MVC days, the phrase was "thin controllers, fat models."

In other words, the reducers aren't listening to actions made specifically for them, i.e. setters. They're aware of the whole app. That's really the key to this entire system. Or at least one of the keys. 

Tame a closer closer look:


**This (correct):**

```js
const sidebarOpen = (state = false, action, types) => {
  switch(action.type) {
    case types.HOME:
    case types.FEED:
    case types.PROFILE:
      return false,
    case types.SETTINGS:
    case types.ACCOUNT:
      return true
    default:
      return state
  }
}
```

**vs (wrong):**

```js
const sidebarOpen = (state = false, action, types) => {
  switch(action.type) {
    case types.SIDEBAR_CLOSE:
      return false,
    case types.SIDEBAR_OPEN:
      return true
    case types.SIDEBAR_TOGGLE:
      return !state
    default:
      return state
  }
}
```

The latter requires:

- **3 additional action creators**
- **additional dispatches in response to user-triggered events**

The first one is just a "smarter reducer" that listens to the *structure-providing actions* of your app. In Respond, we don't dispatch actions carelessly. **Each action** is a meaningful **beacon** or **guiding light** that the rest of the app listens to. This is likely what your app has been missing. 

Everything you need happens with just 1 or 2 actions. Also note: routes with no data requirements only require one action to be dispatched. 

Of course there will be exceptions. There's always exceptions to the rule. But it's very rare that you need a 3rd action in a transition.

**If you understand this pattern, you're well on your way to being dangerous with Respond.** The next thing you'll want to understand, however, is the difference in approaches between Respond's `<Route />` component and what you're likely used to from React Router. Read the [<Route /> component](./route-component.md) docs next.



## `onComplete`

Lastly, the `complete` middleware gives you one final place to tie up any loose ends. Most of the time you won't need it. It works like the rest. It's available to insure there's a patten among Respond apps for anything that might happen after thunks. 

Use it how you please.



## Summary

In closing, the way to think about your callbacks is that each one is designated for a certain sort of work:

- `beforeLeave` is primarily about blocking route transitions
- `beforeEnter` is exclusively about access controls and verification
- `onLeave`exists for cleanup and to provide symmetry (i.e. it's paired to `beforeLeave`)
- `onEnter` is exclusively for DOM-related side-effects
- `thunk` is exclusively for data fetching
- `onComplete` is exclusively for DOM-related side-effects in relation to the 2nd time the page updates *(i.e. due to changes triggered by `thunk`)*


With this setup, you're going to start thinking of the business needs of your app again, and get less microscopically absorbed in components that only complete part of the picture. The route primitive is a place to think of and orchestrate non-rendering aspects. It minimizes component lines of code to that of a traditional rendering library.

Where you want to be--as much as possible--is *using components for local state*, ***not side-effects***. Then Respond state drives them.


> To learn more about optimum component usage with Respond, check [Respond Component Patterns](./respond-driven-patterns.md).

Happy coding, first Responders.