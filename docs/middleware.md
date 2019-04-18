## Pathless Routes

There are many things you can use *pathlessRoutes* for. Having a single convention for both kinds of routes saves you from the disorganized mess that Redux actions/thunks/sagas/epics/etc often becomes.

If you're used to dispatching a dozen state changes for a single user-initiated event (aka using actions as "setters"), be prepared to unlearn some bad habits. 

In addition to greatly reducing the size of your codebase and giving it some much needed structure, there's noticeable performance gains in an app with less reactive changes.

> 💡 Co-locating groups of related routes *(and async work)* makes your intentions clear. And most importantly, coupling data fetching to routes--as part of a *dual action* pattern--means you have less action creators to manage.

> 💡 Linear side-effects, as opposed to random discovery in your component tree, means there are no surprises and gives you greater confidence over the predictability of your app. This approach makes it very natural, for example, to insure a DOM related event, such as scroll restoration, occurs after ***all async work*** is done and the page is updated. Question everything, most of all question if everything you've been taught about React really makes sense to do at the component level, or perhaps within another abstraction such as routes 🕵


## Shadow Routes

"Routes" in Respond don't need to have a `path` property and result in a URL changing in the address bar. Shadow Routes, serve the purpose to "shadow" a real route changing route. They let you setup and perform important async work before a route change can be determined. The primary use case is forms, where you must submit data, validate the response on the server, and only then dispatch to a new route.  

Of course you can use them for any async needs you have like thunks, sagas, or epics. What they provide, in conjunction with regular routes, is a uniform structure, aka *"convention."* Seeing related routes (of both kinds) grouped together in a flat routes map, as simple as it is, has been something missing from Redux development. 


## Overview

Respond comes with the following default middleware pipeline:

```js
middlewares: [
  serverRedirect,      
  anonymousThunk,
  transformAction,
  call('beforeLeave', { prev: true }),
  call('beforeEnter'),
  enter,
  changePageTitle,
  call('onLeave', { prev: true }),
  call('onEnter'),
  call('thunk', { cache: true }),
  call('onComplete')
]
```

It also has one 