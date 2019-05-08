# History Mirroring

Respond is the **first general purpose routing solution to successfully sync the browser's** ***hidden*** **history stack to one your app can see**. Until now, apps didn't bother tracking this stack because they knew they couldn't accurately sync it. 

As a result *"web apps"* have been closer to *"web pages"* than they could be. Websites certainly have not been fertile ground for experiences that make use of the user's history, such as wizards and stack navigators. It's why you see horizontally animated navigators almost exclusively on native, and none on web. 

*Until now*, apps could not know where the visitors have been, only where they *are*. And our apps have been far less than what they could be as a result. Respond opens a whole new world of possibilities.


## The Problem

For those who haven't followed this problem space, basically the browser doesn't let you know what URLs you visited (even on a website you manage), when clearly it *itself* knows. For example, if you visited:

```js
[
  'https://www.reddit.com/r/reactjs',           // clicked link to respondframework.com
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',  // clicked link to hacker news
  'https://news.ycombinator.com',
]
```

the browser knows you visited those pages, as demonstrated by pressing the `back` button until you land on Reddit again. 

However, it would be nice if after you visited hacker news, and pressed the back button, your site could at least perceive itself:

```js
[
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
]
```

> It makes sense for security reasons that browsers won't let your app know what other sites your visitors visited, but it **won't even let you know about itself!**


Imagine the things you could do if you always knew exactly where a user was along the track of history entries:

```js
entries: [
  ['https://www.respondframework.com', {}],
  '[https://www.respondframework.com/docs', {}],
  ['https://www.respondframework.com/tutorial', { stateFoo: 'bar' }],
],
index: 2,
length: 3
```

Unfortunately, all the browser gives is information about the current URL and the `length` for the number of entries in that hidden array:

```js
window.history = {
  length: 3,
  state: { stateFoo: 'bar' }
}

window.location = {
  pathname: '/tutorial',
  search: '',
  hash: ''
}
```

And in regards to the ability to manipulate things, the browser gives you barely anything:

```js
history.pushState({ stateFoo: 'bar' }, '', '/next-path')
```

The `history` object has a few other methods, but they all pretty much amount to the same as pushing another URL. You certainly can't rewrite the whole history stack. More on that shortly...


> HONORABLE MENTION: in the past, solutions for stack navigators have tried to maintain history state in the URL itself, for example: `/tutorial/step-1/step-2/step-3`, but this failed for many reasons: *browser back/next buttons are impossible to sync (users can pop right off your own site), SEO is sacrificed since multiple URLs exist for the same page, URLs are messy and hard to remember, state for each entry is non-existent, etc.* **You don't see it around the web because it doesn't cut the mustard.**



## Tail Handling

Let's now examine another key characteristic of the history stack.

Imagine you tap `back` twice:


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
],
index: 0, // you're now at index 0
length: 3
```

And you go from the homepage to a **different next page:**


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/tutorial/step-1',
],
index: 1,
length: 2
```

What has happened?

- **entries on the current index and onward have been erased**
- **the array has shrunk in length**
- **and the latest entry has been replaced with a new URL**

This is the hidden behavior of the browser that's happening under the hood. It makes sense because you've chosen a different path in history this time, and 2 histories can't exist at the same time, just as in real life.

For good measure, let's visit one more URL:


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2'
],
index: 2,
length: 3
```

You get the idea--any solution that hopes to mirror the hidden history stack must mimic this behavior as well.



## Rewriting The Stack

If it's not clear, we're specifying the wishlist a framework such as Respond must fulfill to end developer suffering when it comes the address bar. So let's kick it up a notch and be sure to get everything we want from Santa.

First, let's take inventory of what we already have. In addition to the stored `state`, `length` and `location` object we just learned about, the `history` api includes the following methods:

- `pushState`
- `replaceState` -- similar to `pushState`, except it changes the URL *in place*, without appending to the array
- `go` -- e.g. you can skip 3 entries ahead with `go(-3)` or 2 backward with `go(-2)`
- `back` -- shortcut for `go(-1)`
- `forward` -- shortcut for `go(1)`

In summary, you can push and replace entries, and you can move along the history stack. 

Unfortunately, without knowing what index you're on, `go`, `back`, and `forward` should not be used. Here's why: if you run `history.back()` there's a possibility you send the user back to an external website you do not control.

So in actuality all we have is *push* and *replace*. 

What any wizard or stack navigator needs though is 3 capabilities:

1. **the ability to reset the entire stack**
2. **to move the user to any index of the stack**
3. **to change any entry on the stack**


### 1. Reset

Imagine you could dispatch a command/action that completely rewrites the entry stack:

```js
const index = 2
const direction = -1
const stack = [
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2',
  'https://www.respondframework.com/tutorial/step-3',
  'https://www.respondframework.com/tutorial/step-4',
]

actions.reset(stack, index, direction)
```

That will obviously result in this stack:

```js
entries: [
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2',
  'https://www.respondframework.com/tutorial/step-3', // visitor is here
  'https://www.respondframework.com/tutorial/step-4', // previous page
],
index: 2,
length: 4,
direction: -1, // yea, we also want to know the last page the developer was on, i.e:
prev: 'https://www.respondframework.com/tutorial/step-4'
```

Keep in mind to achieve this, we must go beyond just changing how your app perceives the stack. **We must also change the URLs that the address bar just saw!**

Respond makes precisely this possible. To pull this off, Respond changes the URLs as fast as it can, creating an almost imperceptible change in the address bar.

Let's look at a real life use case--resetting a checkout stack navigator after a purchase is complete:


#### Use Case:

```js
routes: {
  SUBMIT_CHECKOUT: ({ actions }, { payload }) => {
    const { success, reason } = api.post('checkout', payload)

    return success
      ? actions.reset(stack)
      : actions.flashMessage('Checkout submission failed because ' + reason)
  }
}
```

### 2. Jump

In Respond, we renamed `go` to `jump` and imbued it with some additional capabilities:

```js
actions.jump(-2)                                                      // jump to the 0 index
actions.jump('12345')                                                 // jump to entry by its unique key
actions.jump(-2, { params: { foo: 'bar' }, state: { baz: 'boss' } })  // jump and change the URL + state
actions.jump(0, {}, true)                                             // jump by index
actions.jump(0, {}, true, 1)                                          // fake the direction in state
```

And for good measure Respond offers 2 shortcuts:

```js
actions.back() // actions.jump(-1)
actions.next() // actions.jump(1)
```

> `back` and `next` also accept an action-shaped object as an argument similar to `jump`

Why the rename? Because if you're moving one entry, the smart thing to do is use `back` or `next` to make clear the intent. Therefore, if you were ever to use `go`, it's to `jump` :)

In addition, our `jump` really is a big *jump* since you can also change its state, all the URL parts and even the action type. The 2nd argument accepts all the usual suspects: `params`, `query`, `hash`, `basename`, `state`. Leaving out the `type` is customary, but you could also provide it to jump to a route that was not there before.

All 3 actions will never bounce a visitor from your site. They're safe-guarded from that by throwing an exception. Which is an indicator you should use Respond's `canJump` selector:

```js
const MyComponent = (props, { location }, { back, step1 }) => (
  <button onClick={location.canJump(-1) ? back() : step1()}>
    back
  </button>
)
```

`jump` is also used automatically by Respond under certain circumstances:

- when an **identical adjacent** route is pushed or replaced again

For example, if you have the entries: `['/login', '/tutorial']`, and you push `'/login'` again you **won't** get `['/login', '/tutorial', '/login']`. Respond will `jump` *back* instead of push, decrementing the index. 

Respond is smart enough to know the real intent was to go back. #heuristics

This behavior **enhances user experience and rendering performance**:

- prevents users from wondering why they land on the same page after pressing back/next buttons multiple times
- reduces memory used by StackNavigators, especially on Native, where users often crash apps because they keep pushing the same 1-2 pages as they navigate back and forth between, for example, *the artist detail* page, the *album detail* page, the same *artist detail* page, back to the same *album detail* page, and so on. This can happen a lot on mobile apps like Pandora
- users know when they are going back and forth between the same 2 pages in a stack navigator, and they would prefer the stack navigator did too, instead of push a million entries. This keeps the stack small and easy to navigate.
- stack navigators, wizards and carousels often hide the previous page, but still have it rendered. This allows for the speediest return to the given page, and, again, a reduction in memory

In summary, Respond detects when you're going to a previously visited page, but only when it's directly adjacent to the current entry. Because what else could you be doing? They share the exact same URL. They are the exact same page. You were just there! Your intent is to return to where you just were. `jump` is automatically used in these circumstances.

> And yes, you guessed it, Respond strongly positions itself toward solving the suffering of developers seeking correct declarative stack navigators. More on that below.



### 3. Set

Last but not least, you also need the ability to *set* different aspects on a history entry, *without actually re-visiting it.* `set` is like `jump` except the user stays where they are. It's most commonly used on the current entry:

```js
actions.set({ state: { flag: 'something' }, query: { rapper: 'Big Pun' } })
```

It allows you to set changes that skip the async route pipeline and efficiently go straight to the reducer + address bar. 

Here's how you set entries other than the current one:

```js
actions.set({ state: { flag: 'something' } }, -2)
actions.set({ state: { flag: 'something' } }, '12345')  // by entry key
actions.set({ state: { flag: 'something' } }, 0, true)  // by index
```

There's also these shortcuts:

- `setState`
- `setQuery`
- `setParams`
- `setHash`
- `setBasename`

For example: `actions.setState({ flag: 'something' })`

> Check the [History Action Creators](./action-creators.md#history-action-creators) doc for complete info on our history related action creators.

## Use Cases

So if it's not clear, full history control would make trivial--or at least straightforward--a great number of professional use cases. Our "websites" may now actually be capable of earning the title "app." 

Here is an incomplete list of capabilities that will be sure to make any long time web developer salivate:

- **Stack Navigators**
- Navigator Resets
- Reliable Back/Next buttons within your app (that don't need to be aware of where they're going)
- Jumping to previous pages
- Setting state on previous pages
- Rewriting other URLs in the stack
- **Multiple Navigators, Nested Navigators (more on this later)**
- Default Entries for direct visits (e.g. to /checkout/step-4)
- **Amazing control of checkouts and wizards**
- **Reliable back/next animations on mobile**

One thing they say in the industry is that as the quality of tooling increases, so do the expectations for the experiences we're supposed to create with them (which incidentally results in our job taking the roughly same amount of time). Well, it's happened again! Build something with these tools before your bosses and clients expect you to do it quickly lol.


## Interlude: Challenges to Keeping Sync

Very few attempts have been made in the past to sync an *in-memory* data structure to what it thinks the browser contains (mainly little suggestions on StackOverflow). It's basically been considered impossible. As a result, there's no general purpose solution (that we're aware of) at the time of this writing.

> Submit one as an issue, and we'll amend these lines. Until then you can assume the accuracy of these statements.

Respond follows this path to the bitter end. Because we control every aspect of routing, we are able to do things smaller libraries can't do. We're able to know where users went, from *start to finish.* 

The challenge here is that during *non-instant* async transitions, it's very easy to lose track of how the browser sees the stack. Redirects and blocked route transitions throw a major wrench in the system. Rapid usage of the browser's back/next buttons and pipeline cancellations make matters worse. The biggest culprit however is that chrome doesn't perform *all* transitions synchronously.

Then there's also the challenge that sometimes `sessionStorage` isn't availbale, such as when browsers are in "incognito mode." Of course we have a solution for that as well: fallback to storing all info on each and every history state entry. 

And when browser's don't support history at all, we fallback to an in-memory solution that keeps the same URL the entire time. We have chosen to do this instead of a hash based solution like we previously had with [redux-first-router](https://github.com/faceyspacey/redux-first-router).

To give you confidence that Respond has you covered, let's examine the solutionst to the challenges one by one:

## Session Storage

`sessionStorage` is what allows Respond to remember the history stack when visitors click links to other sites and return. Unlike `localStorage`, `sessionStorage` isn't remembered on subsequent visits once the given browser tab is closed. This is a perfect impedence match for our goal of remembering just the current tab's stack of visited URLs.

This is a foundation we can build upon. From here, we need to make sure to never get out of sync. If we always stay in sync, we can just write to session storage the above info, which by the way looks closer to this:

```js
{
  index: 1,
  entries: [
    ['https://www.respondframework.com', { stateFoo: 'bar' }, '12345'], // '12345' is a unique key, used
    ['https://www.respondframework.com/tutorial/step-1', {}, '567378']  // by `set` and `jump`
  ]
}
```


## History Storage Fallback

For browser's that don't support `sessionStorage`, such as browsers in *incognito* mode, we had to think outside the box.

First of all Respond depends on the browser's `history` API being available. And these days it's ubiquitous.

> Well that's not exactly true; as mentioned above, Respond falls back to a an in-memory solution, in which case the problem of syncing the address bar is also removed, dare we say "collapsed." In that case, if a user taps `back`, they're bounced from your site--that's its prime flaw. The secondary flaw is that there are no URLs to copy/paste out of the address bar, besides the initial page.

But let's just assume we have the `history` API, as somewhere between 98% and 100% of your visitors will have it (it's supported all the way down to Internet Explorer 10). The History API lets you access a piece of state for each URL. For example if we're on the first entry above, we'll have this state:

```js
window.history.state === { 
  stateFoo: 'bar'
}
```

What if we could store the entire history stack there like so:

```js
window.history.state = {
  index: 1,
  entries: [
    ['https://www.respondframework.com', { stateFoo: 'bar' }, '12345'],
    ['https://www.respondframework.com/tutorial/step-1', {}, '12346']
  ]
}
```

That way, when a visitor returns to this page from a link he clicked, we can extract this data just as we would from `sessionStorage`!

Well, it turns out you can. What this means is we store the information for *the entire stack* **on each and every entry's `state` object!** 

> the amount of state you can store per entry is [very large](https://stackoverflow.com/questions/6460377/html5-history-api-what-is-the-max-size-the-state-object-can-be)

Imagine the browser sees this secret stack:

```js
[
  'https://www.reddit.com/r/reactjs',
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
  'https://news.ycombinator.com',
]
```

After the user backs out from hacker news, and returns to `/tutorial` on our site, we'll have the following stored in `history.state`:

```js
window.history.state = {
  index: 2,
  entries: [
    ['https://www.respondframework.com', {}, '12345'],
    ['https://www.respondframework.com/docs', {}, '45632'],
    ['https://www.respondframework.com/tutorial', {}, '71956'],
  ]
}
```

Let's now try to break this. Imagine the user goes back one more page to `/docs`, where we have this stored:

```js
window.history.state = {
  index: 1,
  entries: [
    ['https://www.respondframework.com', {}, '12345'],
    ['https://www.respondframework.com/docs', {}, '45632'],
  ]
}
```

**Hey, we're missing an entry!** Yes, but it doesn't matter because, when returning from an external site, we had just the state we needed at the *tail* of the stack. 

> NOTE: we're examining the *tail* as an example, but rest assured that the same is true for the *head* of the stack.

Therefore if the visitor hits back again, we'll correct the state stored for `/docs`:


```js
window.history.state = {
  index: 1, // notice the index is 1 for /docs
  entries: [
    ['https://www.respondframework.com', {}, '12345'],
    ['https://www.respondframework.com/docs', {}, '45632'],
    ['https://www.respondframework.com/tutorial', {}, '71956'], // notice we now have this entry
  ]
}
```

That way if the visitor clicks another external link from *that entry* and returns, `/docs` will once again have all the necessary info. The takeaway is that we are always able to insure the *tail* and *head* have **all** state stored for immediate return visits.

> Respond will of course recognize what `index` you returned on and remove entries including`/tutorial` and subsequent--call it "tail optimization" :)--to once again mirror how the browser sees the stack.

We won't get into all the heuristics, but the head of the stack has similar behavior handling.



## Worst Case Scenario

So far we have covered what most developers think of when it comes to mirroring the hidden history stack. Now let's cover what is the **bulk of the work in actuality.**

The root of the challenge can be broken down into 4 hurdles:

1. **Excessive Popping:** browser back/next buttons can change routes at will, and they can be tapped rapidly
2. **Automatic Jumping:** Respond automatically convert push/replace actions into jumps when re-visiting adjacent URLs, as that's the preferable user experience
3. **Queue Transitions for Chrome:** chrome doesn't push new entries synchronously!! ([this is a bug](https://bugs.chromium.org/p/chromium/issues/detail?id=775391)) Therefore we need to queue them.
4. **Explicit Commits:** route transitions can be long and asynchronous, and there can be changes to transitions midway (redirects + blocking)



To paint the picture, let's take a look at our worst nightmare:

- user logs in from `/login` **(index 5)**
- user arrives at the private `/dashboard` entry **(index 6)**
- user logs out and is redirected to `/` **(index 7)**
- user **repeatedly and rapidly taps the built-in browser back button**
- user hits `/dashboard` and, since no longer logged in, is redirected to `/login`
- `/login` was **index 5**, so Respond automatically *jumps* back instead of pushing a new route; **the user jumped back 2 entries instead of 1!** This is what Respond deems the correct best behavior. More on this shortly.
- `/login` makes an async request before completing, meanwhile the user is still rapidly tapping the back button

Here's what the relevant history state looks like in the `location` reducer: 

```js
state.location = {
  // rest of state left out for brevity
  index: 5,
  length: 8,
  direciton: -1,
  entries: [
    entries: [
      // previous entries left out for brevity
      {
        type: 'LOGIN', // url: /login
        namespace: '',
        params: {},
        query: {},
        state: {},
        hash: '',
        basename: '',
        key: '123456',
        index: 5,
      },
      {
        type: 'DASHBOARD', // url: /dashboard
        namespace: '',
        params: {},
        query: {},
        state: {},
        hash: '',
        basename: '',
        key: '849305',
        index: 6,
      },
      {
        type: 'HOME', // url: /
        namespace: '',
        params: {},
        query: {},
        state: {},
        hash: '',
        basename: '',
        key: '295834',
        index: 7,
      },
    ],
  ],
  prev: {
    type: 'DASHBOARD', // url: /dashboard
    namespace: '',
    params: {},
    query: {},
    state: {},
    hash: '',
    basename: '',
    key: '849305',
    index: 6,
  },
}
```

The solutions to these hurdles rests on 4 pillars:


### 1. Undo Excessive Popping

The first hurdle Respond must deal with is obviously that the user should now be on, say, *news.ycombinator.com* because he/she tapped the back button in excess of 7 times.

Fortunately, Respond is a boss and *doesn't take shit from anyone*. 

> For reader's whose first language isn't English, this means Respond doesn't allow it. 

Respond's pop handler recognizes when the middleware pipeline is **"busy" as a result of a previous pop**. It then immediately **jumps one entry in the opposite direction to undo** this second pop. 

It's as if the pop never happened. *It's as if all 8+ pops, except the first one one, never happened.*

In effect, Respond normalizes all browser back/next button usage to **not honor** overzealous back/next button pressing **until one completes.** One at a time please!

> Respond even knows if the user quickly switches the opposite direction (i.e. tapped the *next* button).


### 2. Automatic Jumping

With that out of the way, let's discuss a topic you should be aware from above: **it's desirable to us for Respond to automatically jump instead of push (or replace) on occasions.** 

So at this point our entries are:

```js
[
  // committed
  '/login',
  '/dashboard',
  '/'
]
```

At `/dashboard` the user is redirected back to `/login` since the user isn't logged in. For one, this is an edge case (though commonly left as a product bug). Usually you don't move across existing history entries only to be redirected, for if you could visit them in the past, you surely could again. **But not if the user logs out.** 

> NOTE: this is one of the many cases a holistic framework comes to the rescue vs. composing small libraries.

Instead of pushing `/dashboard` so the routes become:

```js
entries: [
  '/login',
  '/dashboard',
  '/login'
],
index: 7
```

or replacing so they become: 

```js
entries: [
  '/login',
  '/login',
  '/'
],
index: 6
```

Respond is smart enough to know the real intent was to go back, **which means we need to jump back by 2 entries:**

```js
entries: [
  '/login',
  '/dashboard',
  '/'
],
index: 5
```

And Respond needs to **rewrite** this to the address bar's real history **while the back button is continuously pressed!** In essence, there's 2 versions of reality: ours, and what the browser is pushing for at the request of the user. And as with life, the winner is the one with the stronger sense of reality. Confidence is king. Respond has your back.


### 3. Queue Transitions for Chrome


There's one remaining problem: Chrome handles these pops asynchronously (both on mobile and desktop). And as we all know, Chrome is the most popular browser by far, so this is a very big problem. 

**What problems does the added asynchrony cause us exactly?**

- If any changes (such as blocked routes or redirects) happen before current pops complete, the browser + Respond will get **out of sync**

To remedy this, Respond implements a **queue**, which guarantees subsequent history changes don't occur until the previous one is complete. The queue performs re-tries in short intervals of half an animation frame, 9 milliseconds.

Now we're able to insure no history changes happen that Respond doesn't know about.


### 4. Explicit Commits ("Innocent Until Proven Guilty")

Respond's middleware essentially treats new dispatches (triggered by browser buttons or UI elements) as `requests` to push the given URL. In other words: *"innocent until proven guilty."* 

Respond **normalizes** requests from the browser buttons vs. your UI elements to appear identically to most of the middleware pipeline. 

That way when it comes to time to `commit` the change, it's just a matter of transending early pipeline  exits and **explicitly commiting the change of state at the same time as changes to the browser's history.**

But just getting there is the difficult part. As you've seen from above, browser buttons + native pop handling can result in lots of unintended or less than ideal `history` changes which are tough to capture and normalize.


### Summary

*Respond normalizes history changes to only let ideal changes pass through to state.* 

The thing to keep in mind with all this is that these issues come from the fact that Respond supports **async pipelines**, which is a major challenge, and a *world first* in its solving.

Async pipelines enable things like asynchronous authentication, where it's expected that the route terminates ("blocks") if not validated . Callbacks can `return false` in that case.

Examples:

- if you want to block a user from leaving an incomplete checkout `return false` from `beforeLeave`. 
- if you want to block a user from entering a page that requires age verification, `return false` from `beforeEnter`

Async pipelines also enable redirects to somewhere else such as a login page (as in the above example).

Most routers provide some very basic utilities for such capabilities. But with Respond, these things are first class. Respond is built around being able to do these things in style.

> And believe us, *async piplines is something you want to do*. These are **THE MOST** requested features from [redux-first-router](https://github.com/faceyspacey/redux-first-router) which became the basis of Respond Framework. 


## Multiple + Nested Stack Navigators

Last but not least, we said we'd address the issue of multiple and nested stack navigators. Let's start with a single stack navigator.

A single declarative stack navigator for your whole site is now straightforward:

- take the entries in `state.location.entries`
- power a `StackNavigator` component, presumably something like this:

```js
const MyStack = (props, state, actions) => {
  const { entries, index, direction } = state.location

  return (
    <StackNavigator entries={entries} index={index} direction={direction}>
      {entry => {
        const Component = components[entry.type] || components.default
        return <Component {...entry} />
      }}
    </StackNavigator>
  )
}

const components = {
  HOME: Home,
  LOGIN: Login,
  ETC: Etc,
  default: Loading
}
```

The navigator wouldn't even be responsible for pushing/replacing/etc new screens.



## Hoisted Data Dependencies

History mirroring is a required capability in order for URL-driven data dependencies to be widely realizable. Without it, Respond can't make the claims it makes in regards to the widespread feasability of hoisted data deps.

> OUR CLAIM: Respond makes it so 99.999% of all apps are better served hoisting their data dependencies to top level routing primitives associated with the URL. The chance you're app isn't one such app is slim to none.

Therefore, it's no wonder the React  community gave up on fetching data based on what URL you're visiting--history mirroring, a key pre-requisite, was thought impossible.

To be clear, [redux-first-router](https://github.com/faceyspacey/redux-first-router) didn't have history mirroring and was extraordinarily successful. But there are scenarios (stack navigators, wizards, etc) where it fell short. And for any solution to be *widely realizable* such holes cannot exist.

And to be even more clear, component-level approaches promoted in mainstream React also fell short. But in the inverse way. There the problem is "solved" by throwing linkability out the window. *Sure, you can have a stack navigator, but expect the URL to stay the same the whole time, or have unreliable browser back/next buttons.* That seemed to be the logic there.

> Facebook's Relay in fact is based on hoisting data deps. So it's quite ironic that the React community at large advocates stuffing data deps in your component tree, only to get lost in the reactive labrynth that becomes your app. That is what Hooks and Suspense is about after all--making it so it's even easier to perform data fetching effects in the component tree.

> But who uses Relay anyway? Apollo certainly has a far greater share of users. And Apollo is all about data fetching within nested components. 

So how did we arrive at this state of affairs where fetching data in relation to the URL became an unwelcome option. Our assessment is that it was thought that React apps are too complex and have too many states in order to tie data dependencies to the URL. **But maybe they didn't have the right tools for the job.** 

The problem in actuality was that there was no frictionless way to create URLs for each *state*. 



### Definition of "State"

But before we go on, we must define "state." What is meant is more like a "state" in [xstate](https://github.com/davidkpiano/xstate) and State Charts. 

**Definition from the [xstate docs](https://xstate.js.org/docs/guides/states.html):** 

> *"A state is an abstract representation of a system (such as an application) at a specific point in time. As an application is interacted with, events cause it to change state. A finite state machine can be in only one of a finite number of states at any given time."* 

Coming from the Redux world, you may be thinking that what is meant by *"state"* is what's returned from your reducers. The correct association is actually events, or, in the case of Redux, *actions.* If you've used `xstate`, you know the equivalent of Redux store state is `context`. 

Basically, in *xstate* `a state === a primaryAction`. 

In Respond's case, those primary actions are route-triggering actions. The ones that change the URL. In terms of UI, route-triggering actions are a way of tagging primary points in the experience. 

So what Respond does is provide *uniform resource indicators (URIs)* for the primary *points* of your app. 



### The Point

The next *point* to understand is that your app must be able to set itself up correctly whether that "point" is:

- dispatched as the first action of the app (as in direct visits)
- or when it's arrived at more naturally as part of a sequence of points leading up to it

The latter is the easier case, and what your app is comprised of when using *routerless* Redux.

> If you were ever someone who built an app with Redux, *but initially with no router*, and then had to incorporate URLs later and wished you could just tag certain actions as *URL-changing* actions, you know exactly what we're talking about. Hopefully at the time, you discovered [redux-first-router](https://github.com/faceyspacey/redux-first-router), which Respond evolved from.

That is to say, apps are built every day that don't exist in the browser and don't need to associate UI states with precise URLs. And that's what building a routerless app with Redux is like. If you've ever been in that enviable position, you likely felt far more in control of your app vs. the struggle of coordinating URL state with UI state plus *further* domain state. 

> Respond, by the way, is all about *all 3 forms of state* living under one roof, while letting you transparently build your app as if the address bar doesn't exist. Yet while still giving your app deep linkability and the benefits of best practices and architecture that come along with said uniform resource indicators.

So routerless Redux apps didn't have the problem of direct visits. Most states can only be reached after specific prior states. Rather, most actions are never dispatched unless specific prior actions are dispatched. Therefore, such apps don't have to deal with, say, a direct visit to step 4 of your checkout wizard or stack navigator. They don't have to replay 3 previous URLs in the address bar in order for reducers to be supplying the correct state. They couldn't even if they tried...*until now.*

Building a React Native app without deep links is an example of this enviable position. The app has just a single entry state, and if you're doing you're job, it usually picks up where you left off through [redux-persist](https://github.com/rt2zz/redux-persist) which remembers your last used state. In fact *redux-first-router* was born precisely out of the need to tag actions in a React Native app with URLs to make it deep-linkable without drastically changing the architecture and codebase (god forbid, having to do things like recompose React Router everywhere).


### State Indirection and Sticky Components

We have one remaining pillar that must be understood to prop up this foundation.

As describe in the [Route component](components.md#pro-tip-match-state-instead) doc, traditional routing libraries like *React Router* display components by matching *directly against the URL*, whereas Respond uses the additional indirection of `state` in order to determine what to display.

So in **naive component-based** routing, match determination is made like this:

- `path in address bar => component`

But with Respond, you have an additional point of *indirection*:

- `path =>` **`state`** `=> component`


It's a sutble difference, but the implications for your app are immense. 

Essentially, one is dumb:

- `App = f(path)`

And the other far smarter due to its longer memory:

- `state = reduce(actions); App = f(state)`


So much has been ingrained in React developers in regards to displaying components based on a one-to-one pairing to what's in the URL. So it may not be immediately obvious. But in the Respond World, the URL is just a *suggestion* for what to display. Reducers have the final say, and ultimately provide a lot more flexibility.

What this allows for is *"sticky"* components that stay in state, even as new URLs dispatch actions. For example, a screen could stay consistent behind a modal that is triggered by its own URL, such as: `/modal`.

**After all, *routerless* Redux apps are perfectly capable of displaying whatever they want, as new actions come in.**



### Bringing It All Together


So we now understand:

- A) that our apps are made up of precise "points" with their own uniform resource locators (URLs), but
- B) each URL may not be equipped to setup all state on its own (it's dependent on previous actions/URLs), and
- C) URLs of course must be able to be visited directly *(because after all, any URL visited once can be shared by a user)*

So we need a natural solution to handle direct visits.

In the [redux-first-router](https://github.com/faceyspacey/redux-first-router) days the best we could aspire to was redirects to the closest most natural route. And that's still an option today.

But that sacrificies much juicy context that links are supposed provide. And worse, under such conditions, we can't build Stack Navigators because browser back/next buttons are not reliable. 

So without further ado, the solution is `route.initialEntries`, an option to rewrite the history stack on direct visit to such a URL. For example on direct visit to: `/checkout/step-4`, we could rewrite the stack to:

```js
route.initialEntries = [
  '/checkout/step-1',
  '/checkout/step-2',
  '/checkout/step-3',
  '/checkout/step-4',
]
```

> And of course there's a function form for complex scenarios. Check [advanced route config](./advanced-route-config.md) for complete reference.

This insures reducers are in their appropriate places, and it even insures any data dependencies from all routes are fetched.

**And that's why history mirroring is required for URL-driven data dependencies (aka "hoisted data deps") to reign supreme once again.**

So our recommendation is to think twice about all the data fetching you're spreading and nesting throughout your component tree, and prepare to unlearn no longer needed *worst practices*. You're making mazes out of your apps and you will never find your way out. Give Respond a try if you're unsure. There's only one way to find out if what we speak is truth.



### Other Strategies

Along with redirects, there is several other solutions when you don't require the complexity of Stack Navigators:

- nested modules (data can be fetched for initial entrance of any route within a given module)
- inherited callbacks (you can share callbacks across multiple similar routes that have the same data deps)
- and redirects as mentioned above

Together along with having an asynchronous pipeline, these strategies provide the fertile ground necessary for URLs to drive your app instead of arbitrary hard to find lifecycle handlers or hooks in your components which inevitably will have you spinning in circles debugging. Hooks are great, but don't need to take over your app. From Respond's standpoint, they are the "low level API." Save them for when you truly need them.



## Final Thoughts

This obviously was an important article in terms of *Respond theory.* It's not necessarily important *in practice*. Novice developers can disregard it if it doesn't make complete sense. More advanced developers can stride more confidently, knowing what's going on under the hood and why we chose the approach we chose.


**Meditation:**

*your routes embody horizontal segways between separate component tree renderings*

*you're no longer shackled by your host environment, the browser*

*you can treat commits to the address bar no differently than commits to state*

*you have the space before and after to seamessly do what you like, workarounds not required.*