# Respond Component Patterns

Respond's most **important** role--even more than routing and modularity--**is data fetching**. Most Respond apps do 100% of their data fetching at the route level, rather than as part of component side-effects. 

The predictability this strategy provides will greatly increase your productivity. Prepare to unlearn what you know regarding using components to do do such work.

> Respond is also the perfect place to coordinate other side effects such as scroll restoration, resize/mouse event handlers, etc, but it's less opinionated about it. There are perfect use-cases for such DOM effects in components using `useEffect`.



## Store-Driven Components

The key to using a combination of store-connected components and standard React components is knowing where to draw the line between the two. In other words: how to expose the correct interface on *dumb React components* so *smart Respond components* can ***"drive"*** them .

Driving dumb React components is based on 4 pillars:

1. moving data-fetching to the route level
2. passing simple values as props that tell the dumb component what to do
3. dumb components only know about local state; smart parents bring in data from the store
4. ***reliable*** *transfer of control from external props to internal state*

The last point is vital and can be addressed in several ways. Taking the token example of a `Modal`, let's examine the various ways this can be achieved.

But first let's agree on what our *requirements* are:


**Requirements:**

- you have a `<Modal />` component
- `visible` state is managed locally by the component (e.g. `useState`) 
- `props.visible` comes from store state **outside**, which *"drives"* it


Here's the **interface** we seek:

```js
components: {
  App: () => (
    <>
      <SmartRespondModal /> // occasionally overlayed over rest of the app
      <RestOfApp />
    </>
  )
}

const SmartRespondModal = (props, state, actions) => (
  <Modal
    visible={!!state.modalText}
    text={state.modalText}
    onClose={actions.cancel}
    onSubmit={actions.confirm}
  />
)
```

Let's checkout 4 ways we can approach this:


## 1) `useMemo` Strategy

```js
import React, { useState, useEffect, useMemo } from 'react'

const Modal = React.memo(({ visible, text, onClose, onSubmit }) => {
  const [show, setVisible] = useState(visible)
  useEffect(() => setVisible(visible), [visible])

  return useMemo(() => {
    if (!show) return null

    const close = () => {
      setVisible(false)
      onClose && onClose()
    }

    const submit = () => {
      setVisible(false)
      onSubmit && onSubmit()
    }
    
    return (
      <div>
        <span onClick={close}>
          x
        </span>
        <h1>{text}</h1>
        <button onClick={submit}>SUBMIT</button>
      </div>
    )
  }, [show, text, onClose, onSubmit])
})
```

Going back to our 4 pillars, the key element is the `visible` prop:

1. the outter `SmartRespondModal` can get the value of this prop from anywhere, perhaps from fetching some data
2. it's a simple value
3. it becomes part of local state, whereas the parent received the data from the store
4. there's *"reliable"* transfer of control from external props to local state using `useEffect` and `useState`


There is one improvement we can make though--if `props.visible` changes, *it will run one more time than necessary*:

- 1st: when `props.visible` changes
- 2nd: when `useEffect` calls `setVisible` after the previous render

Effectively what we have done is turned these lines:

```js
const [show, setVisible] = useState(visible)
useEffect(() => setVisible(visible), [visible])
```

into a thin management layer. This isn't ideal, but it is standard hooks usage. Below we provide a strategy to avoid it. 

This however suffices since the heavy work within `useMemo` is run only a single time (the 2nd time when the value of `show` has changed).


## 2) `React.memo` Strategy

The following strategy is nearly identical to the previous, except instead of `useMemo` we created an additional inner component, wrapped in `React.memo`. 

```js
const Modal = React.memo(({ visible, ...props }) => {
  const [show, setVisible] = useState(visible)
  useEffect(() => setVisible(visible), [visible])

  return <ModalInner show={show} setVisible={setVisible} {...props} />
})

const ModalInner = React.memo(({ show, setVisible, text, onClose, onSubmit }) => {
  if (!show) return null

  const close = () => {
    setVisible(false)
    onClose && onClose()
  }

  const submit = () => {
    setVisible(false)
    onSubmit && onSubmit()
  }

  return (
    <div>
      <span onClick={close}>
        x
      </span>
      <h1>{text}</h1>
      <button onClick={submit}>SUBMIT</button>
    </div>
  )
})
```

Peformance is likely very similar, and it's a matter of preference. If we had to guess, this would be faster because it tags a whole component as memoized, rather than an arbitrary chunk of jsx. That makes for a clearer indicator for React to make optimizations internally.

We also prefer this approach because it makes better separates concerns:

- `Modal` clearly is a thin management layer of the primary variables, making use of hooks
- `ModalInner` is an even dumber--though, performance wise, heavier--container of plain jsx


## 3) `getDerivedStateFromProps` Strategy

The following approach uses a `class` component and `getDerivedStateFromProps` to achieve the same as above. 

Interestingly, a second bit of local state, `propsVisible`, is required to track incoming changes from the outside, *which may or may not match the local state version*. In other words, it's used to keep *sync.*

```js
class Modal extends React.Component {
  constructor(props) {
    super(props)
    this.state = {
      visible: props.visible,
      propsVisible: props.visible
    }
  }

  static getDerivedStateFromProps(props, state) {
    if (props.visible !== state.propsVisible) {
      return {
        visible: props.visible,
        propsVisible: props.visible // a little bit wonky to duplicate `visible` prop, but its stable
      }
    }

    return null
  }

  close = () => {
    this.setState({ visible: false })
    this.props.onClose && this.props.onClose()
  }

  submit = () => {
    this.setState({ visible: false })
    this.props.onSubmit && this.props.onSubmit()
  }

  render() {
    if (!this.state.visible) return null

    return (
      <div>
        <span onClick={this.close}>
          x
        </span>
        <h1>{this.props.text}</h1>
        <button onClick={this.submit}>SUBMIT</button>
      </div>
    )
  }
}
```

This is similar to the ["reset uncontrolled component with a prop"](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#alternative-1-reset-uncontrolled-component-with-an-id-prop) strategy described in the React blog.


The benefit of this approach is that `render` will only ever run once, and because of that we consider it more professional. 

> In non-Respond apps, additional rendering eventually has visible effects on performance. Animation heavy apps, particularly on React Native, are known for such jank. In non-animated apps (which are most common), such bad practices often go unnoticed, which is why so much of the React community lets this stuff fly.

The additional renders also make apps harder to debug. You ever wondered why your `console.logs` were being called more times than expected? And as you begin to hunt around your code as if it's a maze, you wonder why important things are being called when they shouldn't be??

More than likely, you have spent too much of your life in that position. Respond curtails these areas where React is rough around the edges.


## 4) `props.key` Strategy

Which brings us to our preferred approach, known as the ["fully uncontrolled component with a key"](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#recommendation-fully-uncontrolled-component-with-a-key) strategy:

```js
const MyRespondModal = (props, state, actions) => (
  <Modal
    key={!!state.modalText} // *this is key
    visible={!!state.modalText}
    text={state.modalText}
    onClose={actions.cancel}
    onSubmit={actions.confirm}
  />
)

const Modal = React.memo(({ visible, text, onClose, onSubmit }) => {
  const [show, setVisible] = useState(visible)
  if (!show) return null

  const close = () => {
    setVisible(false)
    onClose && onClose()
  }

  const submit = () => {
    setVisible(false)
    onSubmit && onSubmit()
  }

  return (
    <div>
      <span onClick={close}>x</span>
      <h1>{text}</h1>
      <button onClick={submit}>SUBMIT</button>
    </div>
  )
})
```

This renders only once too. It's also the simplest, lacking both `useEffect` and `getDerivedStateFromProps`. By changing the `key` prop, a new `Modal` instance is created, and the previous one destroyed. 

Its downside is that if you hash on an incorrect value passed as `key`, then new instances will be spawned at incorrect times. In the above example, a better key is the `modalText` itself:

```diff
<Modal
-  key={!!state.modalText}
+  key={state.modalText}
   visible={!!state.modalText}
   text={state.modalText}
   onClose={actions.cancel}
   onSubmit={actions.confirm}
/>
```

Now a different modal that is also in the `visible` state will properly trigger the destruction of the previous `Modal` and the creation of a new one. 

Before, changing the `modalText` would not destroy the original `Modal` instance. If you had been animating the appearing/disappearing of your modal, you wouldn't be able to trigger that. By properly timing the destruction/creation of component instances with a fitting `key`, we can achieve things like enter/leave animations:

```js
const SmartRespondModal = (props, state, actions) => (
  <Transition enter={{ opacity: 1 }} leave={{ opacity: 0 }}>
    <Modal
      key={state.modalText}
      visible={!!state.modalText}
      text={state.modalText}
      onClose={actions.cancel}
      onSubmit={actions.confirm}
    />
  </Transition>
)
```


## Takeaway

In the previous examples, the takeaway may be obscured by the simplicity of the implementation. 

The takeaway is that all the complexity surrounding getting the `modalText`--which may be fetched asynchronously over the network--is completely removed from the equation. 

Separation of concerns enables `SmartRespondModal` to be *smart*, but by doing very little work (i.e. only the work of passing state + actions). The real work is offloaded to callbacks (sagas, graphql, etc) on your `route`, and to your reducers. In each case, the work they do is straightforward.

What you save yourself from is:
 
- entanglement coming from both fetching data and manipulating modal states at the component level
- funny business regarding lifting state and context to get appropriate data back up to a top level modal

Of course portals could be used for the latter issue, but there are times when they are not an option. Being able to *beam* state around--often provided via hoisted route level fetching--creates scenarios where single principle components can do their job with little knowledge of the rest of your app. This is what being *"driven"* by store state means.

