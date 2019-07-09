# Co-located Components

Colocated components fulfill the same basic premise as they originally did in Relay, and similar to Apollo. 

But Respond has a different set of constraints: **dependencies must be determined by the URL.**

The way co-located components work in Respond is actually quite ingenius. A parent module loads *one*, and since the `load` callback is *always called first*, all of the child's callbacks get a chance to fire. Then child components are guaranteed to have their fetched data.

The trick is to just create a route with the reserved name **"entry"** and all of its callbacks will be merged into the parent route:



```js
export default createModule({
  routes: {
    entry: {
      thunk: ({ api }, { params }) => api.get(`items/${params.category}`)
    } 
  }
})
```

or graphql:

```js
export default createModule({
  routes: {
    entry: {
      graphql: (req, { params }) => `
        items(category: "${params.category}") {
          id
          name
        }
      `
    } 
  }
})
```


*Here's all that is required of a parent:*

```js
export default createApp({
  routes: {
    LIST: {
      path: '/list/:category',
      load: () => import('./colocated-component')
    }
  }
})
```


Here's how you parameterize the callbacks used:

```js
export default ({
  deps,
  callbacks: {
    beforeEnter = 'beforeEnter',
    thunk = 'thunk',
    onEnter = 'onEnter'
  }
}) => createModule({
  routes: {
    entry: {
      [beforeEnter]: [verifyAccess, saveScroll],
      [thunk]: ({ api }) => api.get(`items/${deps.cat}`),
      [onEnter]: restoreScroll
    } 
  }
})
```

That way the callbacks can be renamed to what's expected by the parent's middleware.

Here's how you dynamically load this module from a parent, passing in the correct arguments:

```js
export default createApp({
  routes: {
    LIST: {
      path: '/list/:category',
      load: async (request, action) => {
        const module = await import('./colocated-component')
        
        return module.default({ 
          callbacks: {
            beforeEnter: 'authenticate',
            onEnter: 'restoreScroll'
          },
          deps: {
            cat: action.params.category
          }
        })
      }
    }
  }
})
```

Since your module this time is a factory of modules, it can be parameterized, **gauranteeing callbacks are called in the current pipeline.** 

Also notice how the child module is paremterized based on the `request` and `action` parameters. This is the secret sauce to child modules generically plugging into 3rd party apps that will most definitely be using different names. 

This is possible because the module is loaded during an ongoing request.

Last but not least, here's how you pair components:

```js
export default createModule({
  components: {
    ColocatedComponent
  },
  routes: {
    entry: {
      thunk: ({ api }, { params }) => api.get(`items/${params.category}`)
    } 
  }
})
```

*The connection made in the parent:*

```js
export default createApp({
  components: {
    App: (props, { location }) => {
      const Component = location.components.list.ColocatedComponent
      return Component ? <Component /> : <Spinner />
    }
  },
  routes: {
    LIST: {...} // namespace 'list' created here
  }
})
```

Parents can use the real name of the child component because because they are protected from collisions by virtue of having created the **namespace** themselves. In this case: `'list'`.


Of course you're not required to render a component from the child. The child doesn't even need to have components. 

You can even have multiple components as usual.

And if you want to get fancy, nothing's stopping you from *not rendering* one initially, and only rendering it later after another route/action. 

**In all cases, it's assumed to have everything it needs by virtue of having loaded.** 

However, the primary use case for *entry routes* is pairing a single component to a single set of callbacks. This is essentially *hoisting* the data (and fx) dependencies to the route level, similar to Relay patterns.

> But in our case completely done at runtime.

With Respond, you're in full control of how you make the pairing. It's like dynamically importing any other child, with the only difference being usage of the `'entry'` keyword. 

In good framework design, features often *collapse* in on itself like this. In other words, this API is a natural extension of how the framework *already works.* You don't have to learn anything new. *Colocated components just work.*

In fact, you could make your whole app out of nothing but co-located components. Wherever you see a *primary page* component, you're guaranteed to see a module pairing it to a set of `entry` callbacks.


## Additional Routes

You can also have other routes alongside `entry`:

```js
export default createModule({
  components: {
    PsychicsList,
    SelectedPsychic
  },
  routes: {
    entry: {
      thunk: ({ api }, { params }) => api.get(`psychics/${params.category}`)
    },
    item: {
      thunk: ({ api }, { params }) => api.get(`psychic/${params.slug}`)
    } 
  },
  reducers: {
    psychics,
    currentPsychic
  }
})
```

In that case, it's beginning to look like a regular Respond module. 

The difference between co-located components and modules is on a *spectrum*. 

The point is the `entry` route provides a natural mechanism to insure a given module has all of its required dependencies before any of its components can be displayed (and before subsequent routes/actions are triggered). 

*What goes in that module is entirely up to you.*

Consequently here's a more appropriate strategy to toggle child component display:

```js
export default createApp({
  components: {
    App: (props, state) => {
      const { psychics, currentPsychic, location } = state
      const { PsychicsList, SelectedPsychic } = location.components.list

      // check that reducer state is filled with data from async data fetching callbacks
      return psychics
        ? currentPsychic ? (
            <>
              <PsychicsList />
              <SelectedPsychic />
            </>
          ) : <PsychicsList />
        : <Spinner />
    }
  },
  routes: {
    LIST: {
      path: '/list/:category',
      load: () => import('./colocated-component')
    }
  }
})
```

> Checking such state is akin to checking that the `entry` (and `item`) callbacks have completed.

As you can see, you don't need to rely on arguments when using modules within your own app. Primarily in larger teams or if making a module for 3rd parties is it prudent to parameterize things like `action.params.category` using something like `deps.category` instead. You will know when to do it.


## Parallel Callbacks


It's possible that the parent route specifies the same callbacks as the child:

```js
export default createApp({
  components: {
    App
  },
  routes: {
    LIST: {
      path: '/list/:category',
      load: () => import('./colocated-component'),
      beforeEnter: [doSomething]
    }
  }
})
```

*Child module:*

```js
entry: {
  beforeEnter: [verifyAccess, saveScroll],
} 
```

To allow for this scenario, Respond merges all 3 callbacks, calling the child module's 2 in parallel to the parent's.


## Co-located Middleware

It's also possibe for child modules to bring their own middleware.

So instead of parameterizing callbacks like this:


```js
export default ({
  deps,
  callbacks: {
    beforeEnter = 'beforeEnter',
    thunk = 'thunk',
    onEnter = 'onEnter'
  }
}) => createModule({
  routes: {
    entry: {
      [beforeEnter]: [verifyAccess, saveScroll],
      [thunk]: ({ api }) => api.get(`items/${deps.cat}`),
      [onEnter]: restoreScroll
    } 
  }
})
```

and relying on parent modules to have compatible middleware, the child module can just bring its own:

```js
import { call } from 'respond-framework/middleware'

export default ({ deps }) => createModule({
  routes: {
    entry: {
      beforeEnter: [verifyAccess, saveScroll],
      thunk: ({ api }) => api.get(`items/${deps.cat}`),
      onEnter: restoreScroll
    } 
  },
  options: {
    middleware: [
      call('beforeEnter'),
      call('thunk', { cache: true }),
      call('onEnter')
    ]
  }
})
```

Internally Respond's middleware runner will branch off to run the child module's middleware, and then complete the parent's pipeline subsequently.


## Conclusion

Callbacks are guaranteed to run because they are merged at just the right time into a route that's already executing. In other words, **the plane is being built while it's flying.**

Co-located components will only be ever be available along side its route dependencies. Even if you navigate to another route--and the component is still showing--it will still have access to all the state from when it first loaded. 

"Co-located components" are just another term for Respond modules with a single `entry` route.

It's a co-location solution that makes perfect sense for Respond. Bet you didn't think co-location and a route centric framework were compatible!

> In the future, as we invest more into GraphQL, we'll investigate the usage of fragments to reduce duplication between parent and child queries, as Relay does.

