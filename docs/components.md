# Components


## Route


Respond's `<Route />` component will appear both familiar and new at the same time:

- it can do the traditional path matching you're likely familiar with from *React Router* and *Reach Router*
- it can do state-based matching, which de-couples what's displayed from the path in the address bar
- *and* it can do some very slick things revolving around the dependency injection of code split components

Let's take a look:

### Traditional Path Matching

Basic path matching like in *React Router* looks like this:

```js
import { Route } from 'respond-framework/components'

<Route path='/settings' component={Settings} />
```

> *Except you never need to specify a `<Router />` ancestor.* You can drop a `<Route />` wherever you want.

It can also do ***composable relative paths*** like *Reach Router*:

```js
const Settings = (props) => (
  <Route path='notifications' component={Notifications} />
)
```

Notice the the absence of a leading slash before `'notifications'`. The `Notifications` component will show when the following path is in the address bar:
- `/settings/notifications`

And the first route will also show on this path...unless we specify the `exact` prop:

```js
<Route path='/settings' component={Settings} exact />
```

Now, it will only show on *exactly* `'/settings'`.


### Introducing the `type` prop

Using paths has a major weakness though: they break in multiple/all places when you want to change them. To remedy that, we recommend using a `type` instead:

```js
const App = (props, state, actions, types) => (
  <Route type={types.SETTINGS} component={Settings} />
)
```

Now you can change the path any time you want in a single place, namely the route:

```js
routes: {
  SETTINGS: '/settings'
}
```

And no matter how many `Route` components in your codebase match `type.SETTINGS`, they will all now match a new URL such as:

```js
routes: {
  SETTINGS: '/dashboard/settings'
}
```


### Pro Tip: Match State Instead

But even the `type` prop is only recommended for temporary prototyping. When your app's needs expand, you use state from reducers:


```js
<Route state={{state => state.page === 'settings'}} component={Settings} />
```

Using reducer state is far more powerful because it makes your main components **"sticky"** while emphemeral components like modals (and the routes that trigger them) come and go:

```js
const pageReducer = (state = 'settings', action, types) => {
  switch(action.type) {
    case types.SETTINGS:
      return 'settings'
    case types.NOTIFICATIONS:
      return 'notifications'
    // case types.MODAL: // routes like /modal/foo will continue to display Settings
    default:
      return state
  }
}
```

What's happened here is matching `state` de-couples the display of components from paths. In traditional naive component-based routes, matching determination is made like this:

- `path in address bar => component`

But with Respond, you have an additional point of indirection:

- `path =>` **`state`** `=> component`


It's a sutble difference, but the implications for your app are vast. 

Essentially, one is quite dumb:

- `App = f(path)`

And the other far smarter due to its longer memory:

- `state = reduce(actions); App = f(state)`


**Respond doesn't forget**, and can continue to display the `<Settings />` component even though your `<Modal />` component is visible:

```js
const showModalReducer = (state, action, types) => action.type === types.MODAL ? true : false

const App = (props, state) => (
  <>
    <Modal visible={state.showModal} />

    // this would match while the modal is showing, and properly appear behind it
    <Route state={state => state.page === 'settings'} component={Settings} />

    // this however would vanish, while the modal shows on top of an empty page
    <Route type={types.SETTINGS} component={Settings} />

    // and the same of course goes for this:
    <Route path='/settings' component={Settings} />
  </>
)
```

Which is why this *subtle change* of additional indirection makes a huge difference in how *you Redux*. The implications are:

- URL changes are just hints at state changes, not the final say
- secondary things can happen without breaking the main things you want to display
- your reducers determine what aspects are sticky and what aspects are not sticky
- each reducer listens to a greater number of actions; they're smarter
- less actions are needed since you don't need additional actions to toggle modals, sidebars, tab bars, navigators, etc ("thin actions, fat reducers")


It's like you built your app with no concerns for routing at all, and just listened to actions as Redux was initially intended. It just works. It's up to you to make your reducers do the right thing, and it's not hard. That's the benefit of a system designed around hopping from one explicit route change to another. 


### Advanced Matching

For advanced matching on different aspects of location state, use the `location` prop. It receives a function which is passed all of the `location` state:

```js
const App = (props, state, actions, types) => (
  <Route
    location={location => location.type === types.SETTINGS && location.query.foo}
    component={Settings}
  />
)
```



In fact, you can combine all forms of matchers:

```js
const SettingsModal = (props) => (
  <Route
    path='/settings'
    type={types.MODAL}
    location={location => location.ready}
    state={state => state.settingsModalText}
    component={Settings}
  />
)
```
This component will only show:

- on the `/settings` path
- when the current type is `MODAL`
- when all async work is done
- and there is *modal settings text* available


One common idiom is to add the `location` prop in order to detect if all data is fetched (and fallback to displaying a `Spinner` if not):

```js
const App = (props, state, actions, types) => (
  <Route
    type={types.Settings}
    location={location => location.ready}
    component={Settings}
    fallback={Spinner}
  />
)
```


### Code Splitting & Component Injection

Respond keeps track of all components dynamically loaded via code splitting in location state:

```js
state.location = {
  // ...
  components: {
    Settings: (props, state, actions) => ...,
    Notifications: (props, state, actions) => ...,
    Foo: undefined, // not loaded yet
  }
  //...
}
```

Therefore you want a nice way to fallback to a spinner when they're not yet available:

```js
const Sidebar = (props, state) => (
  <Route
    state={state => state.page === 'settings'}
    component={state.location.components.Settings}
    fallback={Spinner}
  />
)
```

Simply in the absence of the component (i.e. passing `undefined` as the `component` prop), the fallback will display.

This is so common, `<Route />` saves you a bit of work by passing you the `components` hash:

```js
const Sidebar = (props) => (
  <Route
    state={state => state.page === 'settings'}
    component={components => components.Settings}
    fallback={Spinner}
  />
)
```


**You also don't even have to match any part of state or the URL to display a loading indicator:**

**`<Route component={({ Settings }) => Settings} fallback={Spinner} />`**

In that way, `<Route />` is a convenient automated approach to displaying loading states. **This in fact is the idiomatic way for display components from code-split modules.**


You also receive a second argument, containing all of `state`, which you can use to turn `<Route />` into something more like *React Router's* `<Switch />`:

```js
const PageSwitcher = (props) => (
  <Route
    component={(components, state) => components[state.page]}
    fallback={Spinner}
  />
)
```

> Long time users may recall the component `<Switcher />` from the [redux-first-router](https://github.com/faceyspacey/redux-first-router) and [react-universal-component](https://github.com/faceyspacey/react-universal-component) demos.



### Performance

Lastly, notice how we have enjoyed conveniently passing functions *inline*. In order to achieve the utmost in performance with the least amount of fuss, all prop functions are automatically memoized in order to prevent re-renders. 

The caveat is that if you intend on changing the passed in functions, `<Route />` *should update in response*. The `shouldUpdate` prop solves that:

```js
import { useCallback } from 'react'
import { Route } from 'respond-framework/components'

const Switch = (props) => (
  <Route
    state={useCallback(state => state.page, [])} // memoized for life of component instance
    component={useCallback(components => components[props.page], [props.page])} // updates based on page
    fallback={Spinner}
    shouldUpdate
  />
)
```

We have to use `useCallback` in order to prevent unnecessary re-renderings.

Your code now looks more like standard **verbose React code**. As with most things with Respond, *React* is never gone--it's just a layer conveniently built on top of, which you can peel back when needed.



### Manual Matching

The raw truth is our `<Route />`, isn't all that important. [redux-first-router](https://github.com/faceyspacey/redux-first-router) didn't have one, *and didn't need one*. We only incorporated it within Respond to make sure developers of all levels feel comfortable.

> Especially developers whose first language is Javascript/React, who struggle to imagine how to match components without a path matching utility.

Unlike React Router and Reach Router, path matching is an extremely small part of what Respond brings to the table. And, in fact, since **Respond brings routing state into the store**, it's extremely natural to match directly. 

An intermediate Respond developer should feel comfortable matching on reducer state directly. Here's an example of some of the famed `Switcher` component:

```js
const Switcher = (props, { location, page }) => {
  const { components, ready } = location
  const Component = components[page]

  if (!Component || !ready) return <Spinner />

  return <Component {...props} />
}
```

Yes, we realize it **couldn't be any easier and natural.**


### Gotchas

- Without `shouldUpdate` enabled, you can do this:

```js
const App = (props) => (
  <Route path='/settings' component={props => <Settings {...props} foo={123} />} />
)
```

> and not worry about the passed in component re-rendering like you would in React Router, for the reasons described above.

- If you wanted to allow rendering in order to capture a variable from a closure, you would:

```js
const App = ({ foo }) => (
  <Route
    path='/settings'
    component={useCallback(props => <Settings {...props} foo={foo} />, [foo])}
  />
)
```

- `params` matched in the path prop don't do anything:

```js
<Route path='/settings/:parameter' component={Settings} />
```

Firstly, keep in mind you'll be matching existing routes in your routes map, which have their own named parameter segments. So presumably you used the same names. 

But if you didn't, what you used in your routes map will be what's available in state. For example, this route:

```js
routes: {
  SETTINGS: '/settings/param'  
}
```

will result in **`state.location.params.param`** being available in all your components (and callbacks, etc).


Along with this gotcha is the realization that unlike in React Router et al, you are **no longer reaching for props that have matched params**, eg:

```js
<Route
  path='/settings/:parameter'
  render={props => <Foo param={props.match.params.parameter} />}
/>
```

Instead, you have access to them everywhere seamessly and frictionlessly:

```js
const App = (props, { location: { params, query, basename, etc } }) => (
  <Foo param={param} query={query} basename={basename} etc={etc}
)
```

> And not to mention the plethora of rich routing information Respond provides that you won't find in any other solution.


## Link

Creating links in Respond can be done with both path strings and action action objects:

```js
import { Link } from 'respond-framework/components'


const MyComponent = (props, state, actions) => (
  <>
    <Link to='/list/react'>React</Link>
    <Link to={actions.list({ category: 'respond' })}>Respond</Link> // recommended way
  </>
)
```

Given the route: 

```js
LIST: '/list/:category'
```

2 links will be created:

- `<a href='/list/react'>React</a>`
- `<a href='/list/respond'>Respond</a>`


The recommended approach passes an action object that looks like this:

```js
{
  type: 'LIST',
  params: { category: 'respond' }
}
```


This allows you to change your URLs in one central place in the future:

```js
LIST: '/categories/:category'
```

Only use paths if you're new to Respond and just getting in the swing of things, or if you have a really good reason. One such reason is that you can leave the leading slash out to create a path based on the current URL.


For example if the current URL is `/list`, the following:

```js
<Link to='react'>React</Link>
```

will generate a link to `/list/react`.


### Additional Props:

* **down: boolean = false** - if `true` supplied, will trigger linking/dispatching `onMouseDown` instead of `onMouseUp`.

* **shouldDispatch: boolean = true** - if `false` will not dispatch (useful for SEO when action handled in a parent or child element in a fancy way)

* **target: string** - eg: `'_blank'` to open up URL in a new tab (same as standard `target` attribute to `<a>` tags)

* **redirect: boolean = false** - if `true` supplied, will dispatching your action as a redirect, resulting in the current page in the browser history being replaced rather than pushed. That means if the user presses the browser BACK button, he/she won't be able to go back to the previous page that had the link--he/she will go to the page before that. 

* **onClick: Event => ?boolean** - you can provide an `onClick` handler to do anything you want (e.g. play a sound), but if you return `false` or call `event.preventDefault()` it will prevent linking/dispatching just as you may be used to. *TIP: use instead of `shouldDispatch` when you want to dynamically determine whether to trigger the action or not!*

* **...props:** - you can pass any additional props that an `<a>` tag takes, such as `className` and `style`.



## NavLink

`NavLink` adds a few props to `Link` to serve the purpose of lighting up when it matches the URL in the address bar:


```js
import { NavLink } from 'respond-framework/components'

const action = { type: 'LIST', params: { category: 'respond' } }

<NavLink to={action} activeStyle={{ color: 'purple' }}>
  Respond Framework
</NavLink>
```


* **activeClassName: string** - the class applied when the URL and `to` path match

* **activeStyle: object** - the style object applied when the URL and `to` path match 

* **isActive: (state, location) => boolean** - a custom function to determine whether the link is active. 

* **partial: boolean = false** - if `true`, active styles will be applied if the URL is `/foo/bar` and the link is `/foo`. Whereas by default both URLs must exactly match.

* **ariaCurrent: string** - defaults to `'true'` when active. It's for screen-readers. Learn more [here](https://tink.uk/using-the-aria-current-attribute).
