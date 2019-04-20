# Route Config

*Routes* are central to setting up your Respond app. In this section we'll take a look at the things you're most likely to do when building your routes.

## Overview

There are 2 primary kinds of routes:

- ***regular routes with paths***
- ***"pathless routes"*** 

The *former* is used when you want the URL to change:

```js
routes: {
  VIDEO_PROFILE: {                                     
    path: '/video/:slug',
    thunk: ({ api, params }) => api.get(`video/${params.slug}`) // optionally fetch some data
  }
}

// usage:
const VideoButton = (props, state, actions) =>
  <Button onClick={actions.videoProfile({ slug: 'rick-roll' })
```

The *latter* is commonly used when you have some initial async work to perform, such as **submitting a form** ***before*** it's appropriate to redirect the user:

```js
routes: {
  LOGIN_SUBMIT: {
    thunk: async (request) => {                    
      const { api, action, actions } = request
      const { success, message } = await api.post('login', action.payload)

      if (success) return actions.dashboard()
      return actions.flashMessage(message)
    }
  },
  DASHBOARD: '/dashboard'
}

// usage:
<LoginForm onSubmit={formPayload => actions.loginSubmit(formPayload)} />
```

Observe the **dual action pattern**, where when you want something to happen, a *trigger action* is first dispatched, and then a *follow-up action*. Notice the order is reversed for when the route changing action happens between the 2 kinds of routes.

> If you're used to dispatching a dozen state changes for a single user-initiated event (aka using actions as "setters"), be prepared to awaken to the whole new elegance of Respond. 

In addition to specifying your route as an object, there are 2 additional **shortcuts**:


**value as a path:**

```js
FOO: '/foo/:param', 
```

**value as a thunk:**

```js
BAR: ({ api }) => api.get('money')
```



Lastly, there is a ***special case*** where a route has neither a path nor a thunk. Rather, it's used simply as a **namespaced container for nested routes**:

```js
DASHBOARD: {
  routes: {
    ACCOUNT: '/account',
    METRICS: '/metrics',
    PAYMENT: '/payment'
  }
}
```

Or in its more common **code-split module** form:

```js
DASHBOARD: {
  load: () => import('../modules/dashboard')
}
```

*src/modules/dashboard/index.js:*

```js
export default createModule({
  routes: {
    ACCOUNT: '/account',
    METRICS: '/metrics',
    PAYMENT: '/payment'
  }
})
```


The `DASHBOARD` namespace could just have easily had its own path and thunk (as well as other callbacks):

```js
DASHBOARD: {
  path: '/dashboard'
  load: () => import('../modules/dashboard'),
  thunk: ({ api }) => api.get('items'),
  onEnter: () => window.scrollTo(0, 0)
}
```

But because originally it had none of these, dispatching `actions.dashboard()` does nothing. In order to make a change you had to dispatch:

-  `actions.dashboard.account()` 
- `actions.dashboard.metrics()`


For a deep dive on the behavior of module nesting, view the [respond modules](./respond-modules.md) doc.

Let's get to the meat of how you extract `params` (and other things like `query`, `hash`, etc) from the URL:



# Path Params

Having learned from [redux-first-router](https://github.com/faceyspacey/redux-first-router), Respond has all your needs covered when it comes to transforming `params` objects to and from the path part of the URL.

## Basic Usage


*Route:*

```js
POST: {
  path: '/post/:slug'
},
```

*Outgoing Action:*
```js
// raw action:
dispatch({ type: 'POST', params: { slug: 'something-good' } })

// or more commonly:
actions.post({ slug: 'something-good' }) // object interepreted as params

// or:
actions.post({ params: { slug: 'something-good' } })
```

*Incoming Path:*
```js
'/post/something-good'
```

The above path will appear in the address bar if you trigger one of the actions; and if a user visitors that path directly, the reciprocal actions will be disaptched. 


## Optional Params

If you'd like to make a parameter segment optional, append a question mark to the param name:

*Route:*
```js
POST: {
  path: '/post/:slug/:date?'
},
```

*Outgoing Action:*
```js
actions.post({ slug: 'something-good', date: 'february-3-2019' })

// or:
actions.post({ slug: 'something-good' })
```

*Incoming Path:*
```js
'/post/something-good/february-3-2019'

//or:
'/post/something-good'
```

They will both match.


## Transformations 

### `to/fromPath(value, key): string`

To customize the param transformation into how you want your app to see it, here's how the date from above could be handled:

```js
POST: {
  path: '/post/:slug/:date',
  fromPath: (val, key) => key === 'date' ? new Date(val) : val,
  toPath: (val, key) => {
    if (key === 'date') {
      const month = numToMonth(val.getMonth())
      const day = val.getUTCDate()
      const year = val.getFullYear()

      return `${month}-${day}-${year}`
    }

    return value
  }
},
```

Notice you are passed each param one by one, where the `key` is the name of the param.

## Custom encoding/decoding

### `to/fromPath(value, key, encodedVal): string`

```js
POST: {
  path: '/post/:slug',
  fromPath: (val, key, encodedVal) => decodeURIComponent(encodedVal),
  toPath: (val, key, encodedVal) => encodeURIComponent(val)
},
```

The 1st argument, `val`, in both functions is the *application-ready* version of the path segment, decoded with `decodeURIComponent` prior to being passed to `fromPath`.

The 3rd argument, `encodedVal`, in both functions is the *URL-ready* version of the path segment, encoded with `encodeURIComponent` prior to being passed to `toPath`.

If you'd like to perform your own custom encoding/decoding, notice the reverse symmetry:

- **decode `encodedVal` in `fromPath`**
- **encode `val` in `toPath`**

That may seem odd, but keep in mind this *not* the primary use case. This is the "low level" approach. Respond is all about *sensible defaults*, but with a straightforward customization path.

The default is automatic encoding/decoding of the 1st argument, `val`, as in the previous example. 



## `numbers: boolean` *(default: false)*

If you would like the actions generated from the path to automatically convert number strings to real javascript numbers, set `numbers` to `true`:

```js
ARTICLE: {
  path: '/article/:num',
  digits: true
}
```


*Path from direct visit:*
```js
'/article/42'
```

*Resulting Action Automatically Dispatched:*

```js
{ type: 'ARTICLE', params: { num: 42 } }
```

> Numbers with decimals will be converted to floats too.



## `sluggify: boolean` *(default: false)*

Say you want to automatically transform article titles into slugs (and vice versa), Respond has you covered:

```js
ARTICLE: {
  path: '/article/:slug',
  sluggify: true
}
```

*Action:*
```js
const title = article.title // === 'Respond Is The Only Way To React'
actions.article({ slug: title })
```

*Path:*
```js
'/article/respond-is-the-only-way-to-react'
```

> NOTE: you really should store a unique slug in your database yourself though, but this can be useful for prototyping when all you have to deal with is a title.




## Optional Static Segments ("aliases")

If you want to match `'/'` and, say, `'/home'` with the same route, here's how you do it:

```js
HOME: {
  path: '/(home)?'
}
```

If you need more than a 2nd alias, use `:param?`. If you have no use for the parameter and don't want to let just any string match, use regexes:



## Regex Params

Regexes let you require that params match a specific pattern:

```js
HOME: {
  path: '/:param(home|main|foo)'
}
```

That will match:


- `/home`
- `/main`
- `/foo`
- and if you want to match `/`, add a question mark:

```js
HOME: {
  path: '/:param(home|main|foo)?'
}
```

Now say you wanted to match only **numbers:**

```js
ENTITY: {
  path: '/entity/:num(\\d+)'
}
```

That will capture paths with 1 or more numbers like `/entity/42`.


If you want to also capture **missing numbers**, you can use the **asterisk**:

```js
ENTITY: {
  path: '/entity/:num(\\d*)'
}
```

But that will only match `/entity/`.

If you want to match `/entity` **without the trailing slash** you would use the **question mark** outside the regex:

```js
ENTITY: {
  path: '/entity/:num(\\d+)?'
}
```

Complex regexes can be created between the parentheses, but no capture groups using parentheses are allowed. And you must escape slashes in patterns like the digit matcher above `\\d`. **So that means 2 front slashes, not 1!** 



## Multi Segment Params

Now say you want to match file paths like Github does as you browse the files of a repo, that has to be handled in a specific way since the file path has slashes of its own:

```js
REPO: {
  path: '/repo/:user/:repo/blob/:branch/:filePath+'
}
```

An action that matches that will look like this:
```js
{
  type: 'REPO',
  params: {
    user: 'respond-framework',
    branch: 'master',
    filePath: 'src/core/createRouter.tsx'
  }
}
```

And a path that matches will look like this:

```js
'/repo/faceyspacey/respond-framework/blob/master/src/core/createRouter.js'
```

The main ingredient is the `+` suffix.



## Optional Multi Segment Params

If we wanted to make `:filePath` optional, we would use an astersik **(a question mark will not work in this case).**


```js
REPO: {
  path: '/repo/:user/:repo/blob/:branch/:filePath*'
}
```

You could dispatch a matching action without the `filePath` param:
```js
{
  type: 'REPO',
  params: {
    user: 'respond-framework',
    branch: 'master',
  }
}
```

And the resulting URL would be:

```js
'/repo/faceyspacey/respond-framework/blob/master'
```



## *Multiple* Multi Segment Params

And if you really want to go bezerk you can match *multiple* multi segment params like so:


```js
WILD: {
  path: '/wild/:and+/crazy/:kids+'
}
```

Action:
```js
actions.wild({ and: 'foo/bar', kids: 'baz/omar' })
```



## `defaultParams: Object | Function`

To bi-directionally apply default params in the case that they are missing, you can specify them as an object:

```js
FOO: {
  path: '/foo/:param?',
  defaultParams: { param: 'bar' }
}
```

or dynamically with a function:

```js
FOO: {
  path: '/foo/:other/:param?',
  defaultParams: otherParams => ({ param: 'bar', ...otherParams })
}
```

Taking the latter example, and dispatching:

```js
actions.foo({ other: 'abc' })
```

will generate:

```js
'/foo/abc/bar'
```

And inversely, visiting `'/foo/abc'`, will generate:
```js
{ type: 'FOO', params: { other: 'abc', param: 'bar' } }
```

You can perform any sort of logic you want in this function. In addition to the original `params`, you are also passed the `route` and `options` for the given module as 2nd and 3rd arguments:

```js
route.defaultParams = (params, route, options) => ... // do something great
```

You can also use `defaultParams` to tack on additional params that don't affect the URL:

```js
FOO: {
  path: '/foo/:param?',
  defaultParams: { param: 'bar', info: 'geek' }
}
```

# Queries

You likely have never seen a router with query matching capabilities, so let's learn something new.


## Boolean Specifiers

```js
HOME_REFFERED: {
  path: '/',
  query: {
    referrer: true
  }
}
```

This will match a URL like:
```js
'http://www.myawesomesapp.com?referrer=google'
```

but not:
```js
'http://www.myawesomesapp.com'
```

Conversely, you could prevent matching the presence of a key:

```js
HOME_DIRECT: {
  path: '/',
  query: {
    referrer: false,
  }
}
```

If you're wondering whether you can match just a query without the path, the answer is *no*. If that were possible, either multiple actions would have to be dispatched (i.e. when another route matches by path) or the one ordered first would be the only one dispatched. Which is why it's more consistent to think of queries as specifiers for existing URLs. 

In reality, matching on anything other than paths has been uncommon *(non-existent?)* until now. It's likely another *world first* brought to you by Respond Framework--maybe you'll find uses for it not imagined yet. That is what software is all about, eh.


## String Specifiers

Precise values for keys can be matched:

```js
HOME_DIRECT: {
  path: '/',
  query: {
    referrer: 'google',
  }
}
```

Knowing what we now know about how `paths` are always required, and how `queries` are just deeper specifiers, let's see how we could get value of this:

```js
HOME_DIRECT: {
  path: '/',
  inherit: 'HOME',
  query: {
    referrer: 'google',
  },
  beforeEnter: ({ location }) => mixpanel.track('google', location)
},
HOME: {
  path: '/',
  beforeEnter: () => ...,
  thunk: () => ...,
  onEnter: () => ...,
},
```

In this example, `HOME_DIRECT` will inherit non-overriden callbacks from `HOME`, making it unnecessary to write them again. 

> See [Advanced Route Config](./advanced-route-config.md) for more on code re-use between routes.

Also notice how the more specific route, `HOME_DIRECT`, came first. This is essential so `HOME` doesn't match. 

> The order of keys on javascript objects is preserved in all browsers, old and new. In the past, objects with numerical keys were sometimes re-ordered by browsers. That does not apply here. Rest assured, mighty programmer.


## Regex Specifiers

```js
CONFIRMATION: {
  path: '/checkout/thank-you',
  query: {
    order_id: /\d+/
  }
}
```

nuff said.


## Function Specifiers

Don't want your search results page appearing in google for naughty keywords?:

```js
SEARCH: {
  path: '/search',
  query: {
    term: val => isNaughtyWord(val)
  }
}
```


## `to/fromSearch(val, key): string`

Like `toPath` and `fromPath`, you can convert query values. Say you want your query string to appear as upper case in the address bar, but lower case in Respond state:

```js
SEARCH: {
  path: '/search',
  toSearch: (val, key) => value.toUpperCase(),
  fromSearch: (val, key) => value.toLowerCase()
}
```


## `defaultQuery: Object | Function`

In the absence of a given query key/val, you can provide default ones:

```js
HOME: {
  path: '/',
  defaultQuery: { kind: 'direct' }
},
LANDING: {
  path: '/landing',
  defaultQuery: query => ({ intent: 're-target', ...query })
}
```


# Hash

The Hash can be specified via all the same strategies as query strings, but for a single value:

```js
FOO: {
  path: '/',
  hash: true,
}
```

All examples:

```js
hash: false,
hash: 'foo',
hash: /bar/,
hash: val => val === 'merkaba',
defaultHash: 'pantera',
toHash: hash => hash.toUpperCase(),
fromHash: hash => hash.toLowerCase()
```

# Entry State

Respond keeps track of all the entries your visitors visit for the duration of their visit, even if they click a link to another site and return. So we can tell you both the current state, and the state of previous routes along the history *track* (um, stack). 

For Respond `state` is just like queries, but hidden from the URL.

It's also like the kind of *"state"* you're used to setting per component instance, but in the case of Respond, *per route*. See [History Action Creators](./action-creators.md#history-action-creators) for how you can `setState` on the current route or any other route in `state.location.entries`.


## Matching Entry State

Entry state, unlike params, queries and hashes, cannot be set unless you've at least visited a given URL once. Therefore you can't match on it. But you can provide `defaultState` which is automatically set for you when you visit the URL. As mentioned, you can also add state to any action you dispatch.


## `defaultState: Object | Function`

With Respond it makes *great sense* to use route state, rather than component state. Why? 

- so you don't have to prop-drill to pass state around
- *or* even think about lifting when you add features or refactor
- *and* so you don't have to create reducers for basic UI state *(setting state on a current route is a lot like MobX)*

Let's take a look at how `defaultState` can help us with some forms:

```js
ORDER_FORM: {
  path: '/product/:id/order',
  defaultState: {
    amount: 1,
    size: 'large',
    color: 'purple',
    kind: 'pro',
    delivery: 'fedex',
    date: moment().add(7,'days').format('YYYY-MM-DD')
  }
}
```

Imagine your on Amazon.com, or an ecommerce site where you **customize your product** quite a bit. Having state on the route lets you easily get the above values automatically drilled down to perhaps dozens of customization widgets (e.g. checkboxes, radio forms, sliders, datepickers, color pickers lol, you get the idea). Here's how we would handle the above *half dozen:*


**Corresponding Fancy Input Components:**
```js
const AmountInput = (props, { amount }, { setState }) => ( // setState goes straight to reducer
  <input
    type='number'
    value={amount}
    onChange={e => setState({ amount: e.target.value })}
  />
)

const SizeSelector = (props, { size }, { setState }) => ...;
const ColorPicker = (props, { color }, { setState }) => ...;
const KindDropdown = (props, { kind }, { setState }) => ...;
const DeliveryRadio = (props, { delivery }, { setState }) => ...;
const DateWidget = (props, { date }, { setState }) => ...;
```

> `seState` skips the route pipline, and goes straight to Respond's built-in `location` reducer. It's very fast.


Let's say your app has different options per product type, here's how we might address that:


```js
ORDER_FORM: {
  path: '/:category/product/:id/order',
  defaultState: (state, action) => { // defaultState receives the `action`
    return action.category === 'shirts'
      ? { size: 'large', amount: 1, ...state }
      : { size: 10, amount: 1, ...state} // shoes
  }
},
```

> Unlike `setParams`, `setQuery` and `setHash`, `setState` receives the `action` as a second argument, specifically for purposes like this. The 3rd and 4th args are `route` and `options` like the others.

**Complete Contact Form Example:**

Let's complete the thought to see how easy this really is. Here's a complete **contact form** example:


**Routes:**

```js
import { createModule } from 'respond-framework'
import { ContactForm, Thanks } from './components'

const config = {
  components: { ContactForm, Thanks },
  routes: {
    CONTACT_FORM: {
      path: '/contact',
      defaultState: state => {
        const { first = '', last = '' } = state
        const fullName = `${first} ${last}`.trim()
        return { ...state, fullName }
      }
    },
    CONTACT_SUBMIT: { // pathless route
      thunk: async ({ api, location, actions }) => {                    
        const { state } = location // simply get entry state from Respond's location reducer
        const { success, data, message } = await api.post('contact', state)

        if (success) return actions.thanks({ state: data }) // show some ephmeral state on the next page
        return actions.flashMessage(message) // flashMessage also uses entry state under the hood
      }
    },
    THANKS: '/thanks'
  }
}

export default createModule(config)
```

**Components:**


```js
const Input = ({ key }, state, { setState }) => (
  <input value={state[key]} onChange={e => setState({ [key]: e.target.value })} />
)

const TextArea = ({ key }, state, { setState }) => (
  <textarea onChange={e => setState({ [key]: e.target.value })}>
    {state[key]}
  </textarea>
)

export const ContactForm = (props, state, actions) => (
  <form>
    <Input key='first' />
    <Input key='last' />
    <Input key='email' />
    
    <TextArea key='email' />

    <button onClick={actions.contactSubmit}>SUBMIT</button> // form state already stored
  </form>
)

export const Thanks = (props, state) => (
  <div>
    <h1>Thanks {state.location.state.fullName}</h1> // passed by `actions.thanks({ state })`
  </div>
)
```

As you can see, route level state is insanely powerful in its simplicity. Here's some efficiencies it gave us:

- default entry state formatting
- easy access to store state/actions within nested leaf components
- high perf minimal rendering of leaf components *(one of the many benefits of easily accessible "store" state)*
- easy access to entry state within your pathless route thunk
- state to the rescue even for flash messaging + ephemeral info on the *Thanks* page

In this example, there **isn't one bit of cruft.** It's literally **100% business logic**, and no extra work *"because that's how computers must be talked to."*

Cheers. **Don't just React,** ***Respond.***