# History Mirroring

Respond is the **first general purpose routing solution to successfully sync the browser's** ***hidden*** **history stack to one your app can see**. Until now, apps didn't bother tracking this stack because they knew they couldn't accurately sync it. 

As a result *"web apps"* have been closer to *"web pages"* than they could be. Websites certainly have not been fertile ground for experiences that make use of the user's history, such as wizards and stack navigators. It's why you see horizontally animated navigators almost exclusively on native, and none on web. 

*Until now*, apps could not know where the visitors have been, only where they *are*. And our apps have been far less than what they could be as a result. Respond opens a whole new world of possibilities.


## The Problem

For those who haven't followed this problem space, basically the browser doesn't let you know what URLs you visited (even on a website you manage), even though clearly it *itself* knows. For example, if you visited:

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

> It's understandable that browsers won't let your app know what other sites your visitors visited, but it **won't even let you know about itself!**


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

And that's not to mention the browser gives you barely any ability to manipulate besides pushing another state:

```js
history.pushState({ stateFoo: 'bar' }, '', '/next-path')
```

The `history` object has a few other methods, but they all pretty much amount to the same as pushing another URL. You certainly can't rewrite the whole history stack. More on that shortly...


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

You go from the homepage to a **different next page:**


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs/installation',
],
index: 1,
length: 2
```

What has happened?

- **entries on the current index and onward have been erased**
- **the array has shrunk in length**
- **and the latest entry has been replaced with a new URL**

This is the hidden behavior of the browser that's happening under the hood.

For good measure, let's visit one more URL:


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs/installation',
  'https://www.respondframework.com/docs/getting-started'
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
  'https://www.respondframework.com/tutorial',
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2',
  'https://www.respondframework.com/tutorial/step-3',
]

actions.reset(stack, index, direction)
```

That will obviously result in this stack:

```js
entries: [
  'https://www.respondframework.com/tutorial',
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2', // visitor is here
  'https://www.respondframework.com/tutorial/step-3', // previous page
],
index: 2,
length: 4,
direction: -1 // yea, we also want to know the last page the developer was on
```

Keep in mind to achieve this, we must go beyond just changing how your app perceives the stack. **We must also change the URLs that the address bar just saw!**

Respond makes precisely this possible. To pull this off, Respond changes the URLs as fast as it can, creating an almost imperceptible change in the address bar.

Let's look at a real life use case--resetting a checkout stack navigator after a purchase is complete:


#### Use Case:

```js
routes: {
  SUBMIT_CHECKOUT: ({ actions, state }) => {
    const { success, reason } = api.post('checkout', state.checkout)

    return success
      ? actions.reset(stack)
      : actions.flashMessage('Checkout submission failed because ' + reason)
  }
}
```

### 2. Jump

In Respond, we renamed `go` to `jump` and imbued it with some additional capabilities:

```js
actions.jump(-2)                                                      // jump to the 0 index,
actions.jump(-2, { params: { foo: 'bar' }, state: { baz: 'boss' } })  // jump and change the URL + state
actions.jump(0, {}, true)                                             // jump by index
actions.jump(0, {}, true, 1)                                          // fake the direction 
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


### 3. Set

Last but not least, you also need the ability to *set* different aspects on a history entry, *without actually re-visiting it.* `set` is like `jump` except the user stays where they are. It's most commonly used on the current entry:

```js
actions.set({ state: { flag: 'something' , query: { rapper: 'Big Pun' } })
```

It allows you to set changes that skip the async route pipeline and efficiently go straight to the reducer. 

Here's how you set entries other than the current one:

```js
actions.set({ userChose: 'xyz' }, -2)
actions.set({ userChose: 'xyz' }, 0, true)  // by index
```



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

Expectations have been risen for us web developers yet again!


## Manual Syncing

Very few attempts have been made in the past to manually sync an *in-memory* data structure to what it thinks the browser contains (mainly little suggestions on StackOverflow). There's no general purpose solution that we're aware of at the time of this writing. 

> Submit one as an issue, and we'll amend these lines. Until then you can assume the accuracy of these statements.

Respond follows this path to the bitter end. Because we control every aspect of routing, we are able to do things smaller libraries can't do. We're able to know where users went, from *start to finish.* 

The challenge here is that during *non-instant* async transitions, it's very easy to lose track of how the browser sees the stack. Redirects and blocked route transitions throw a major wrench in the system. The primary culprit however is rapid usage of the browser's back/next buttons. 

That's the meat of the work and there are some tough browser quirks we had to overcome.

Then there's also the challenge that sometimes `sessionStorage` isn't availbale, such as when browsers are in "incognito mode." Of course we have a solution for that as well: fallback to storing all information on each and every history state entry. More on that shortly.

> And when browser's don't support history at all, we fallback to an in-memory solution that keeps the same URL the entire time. We have chosen to do this instead of a hash based solution like we previously had with [redux-first-router](https://github.com/faceyspacey/redux-first-router).


## Session Storage

`sessionStorage` is what allows Respond to remember the history stack when visitors click links to other sites and return. Unlike `localStorage`, `sessionStorage` isn't remembered on subsequent visits once the given browser tab is closed. This is a perfect impedence match for our goal of remembering just the current tab's stack of visited URLs.

This is a foundation we can build upon. From here, we need to make sure to never get out of sync. If we always stay in sync, we can just write to session storage the above info, which by the way looks closer to this:

```js
{
  index: 1,
  entries: [
    ['https://www.respondframework.com', { stateFoo: 'bar' }, 12345],
    ['https://www.respondframework.com/tutorial/step-1', {}, 12345]
  ]
}
```


## History Storage Fallback

For browser's that don't support `sessionStorage`, such as browsers in *incognito* mode, we had to think outside the box.

First of all Respond depends on the browser's `history` API being available, as minimal as it is. And these days it's ubiquitous.

> Well that's not exactly true; Respond falls back to a an in-memory solution, in which case the problem of syncing the address bar is also removed, dare we say "collapsed."

But let's just assume we have the `history` API. The History API lets you access a piece of state for each URL. For example if we're on the first entry above, we'll have this state:

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
    ['https://www.respondframework.com', { stateFoo: 'bar' }, 12345],
    ['https://www.respondframework.com/tutorial/step-1', {}, 12346]
  ]
}
```

Well, it turns out you can. What this means is we store the information for the entire stack **on each and every entry's `state` object!** 

> the amount of state you can store per entry is [very large](https://stackoverflow.com/questions/6460377/html5-history-api-what-is-the-max-size-the-state-object-can-be)

Imagine the browser sees this secret stack from above:

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
    ['https://www.respondframework.com', {}, 12345],
    ['https://www.respondframework.com/docs', {}, 45632],
    ['https://www.respondframework.com/tutorial', {}, 71956],
  ]
}
```

Imagine now the user goes back one more page to `/docs`, where we have this stored:

```js
window.history.state = {
  index: 1,
  entries: [
    ['https://www.respondframework.com', {}, 12345],
    ['https://www.respondframework.com/docs', {}, 45632],
  ]
}
```

**Hey, we're missing an entry!** Yes, but it doesn't matter because, when returning from an external site, we had just the state we needed at the *tail* of the stack. 

> The same is true for *head* of the stack.

Therefore if the visitor hits back again, we'll correct the state stored for `/docs`:


```js
window.history.state = {
  index: 1, // notice the index is 1 for /docs
  entries: [
    ['https://www.respondframework.com', {}, 12345],
    ['https://www.respondframework.com/docs', {}, 45632],
    ['https://www.respondframework.com/tutorial', {}, 71956], // notice we now have this entry
  ]
}
```

That way if the visitor clicks another external link from *that entry* and returns, `/docs` will once again have all the necessary info. Respond will of course recognize what `index` you returned on and remove `/tutorial`--call it "tail optimization"--to once again mirror how the browser sees the stack.

We won't get into all the heuristics, but the head of the stack has similar behavior handling.



## Redirects, Blocking, Async Transitions and Rapid Back/Next Tapping

So far we have covered what most developers think of when it comes to mirroring the hidden history stack. Now let's cover what in actuality is the bulk of the work.

The root of the challenge can be broken down into 4 aspects:

1. that route transitions can be long and asynchronous
2. and there can be changes to transitions midway (redirects + blocking)
3. browser back/next buttons can change routes at will, and they can be tapped rapidly
4. **chrome doesn't push new entries synchronously!!** ([this is a bug](https://bugs.chromium.org/p/chromium/issues/detail?id=775391))


To paint the picture, let's take a look at our worst nightmare:

- user is on a private page, `/dashboard`, on index 6
- user logs out and is redirected to `/home`
- user is now on index 7
- user **repeatedly and rapidly taps the built-in browser back button**
- user hits `/dashboard` and, since no longer logged in, is redirected to `/login`
- `/login` was index 5, so Respond automatically pops back instead of pushing a new route *(this is a powerful respond capability; more on this soon)*
- `/login` makes an async request before completing, meanwhile the user is still rapidly tapping the back button

Here's what our history state looks like (for the first time showed from the perspective of our location reducer): 

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
        key: 123456,
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
        key: 849305,
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
        key: 295834,
        index: 7,
      },
    ],
  ]
}
```


The first hurdle Respond must deal with is obviously that user should now be on, say, *news.ycombinator.com* because he/she tapped the back button in excess of 7 times.

Fortunately, Respond is a boss and doesn't take shit from anyone. Respond's pop handler recognizes when the middleware pipeline *is "busy" as a result of a previous pop* (aka index change based on browser back/next button usage). It then immediately jumps one entry in the opposite direction to undo this second pop that just happened. 

It's as if the pop never happened. It's as if all 8+ pops, except the first one one, never happened.

In effect, Respond normalizes all browser back/next button usage to *not honor* overzealous back/next button pressing. One at a time please!

Respond even knows if the user quickly switches the opposite direction by tapping the *next* button instead--if the pipeline is not complete, Respond is not having it!

With that out of the way, let's discuss how Respond handles dispatching a route adjacent to the current possition. So above we have:

```js
[
  // ommitted
  '/login',
  '/dashboard',
  '/'
]
```

At `/dashboard` this time around, since the user isn't logged in, the user is redirected back to `/login`. For one, this is already an edge case--usually you don't move across existing history entries only to be redirected, for if you could visit them in the past, you surely could again. But not if the user logs out. 

> This is one of the many cases a holistic framework can help you with that you absolutely will get no help for when composing small libraries.

Secondly, instead of pushing `/dashboard` so the routes become `['/login', '/dashboard', '/login']` or replacing so they become `['/login', '/login', '/']`, Respond is smart enough to know the real intent was to go back. #heuristics

This behavior solves several things:

- users wondering why they land on the same page after pressing back/next buttons multiple times
- optimum StackNavigators, especially on Native, where users often crash apps because they keep pushing the same 1-2 pages as they navigate back and forth between, for example, *the artist detail* page, the *album detail* page, the same *artist detail* page, back to the same *album detail* page, and so on. This can happen a lot on mobile apps like Pandora.
- users know when they are going back and forth between the same 2 pages in a stack navigator, and they would prefer the stack navigator did too, instead of push a million entries

So Respond has made the decision to detect when you're going to a previously visited page, but only when it's directly adjacent to the current entry. Because what else could you be doing? They share the exact same URL. They are the exact same page. You were just there! Your intent is to return to where you just were. 

> And yes, you guessed it, Respond strongly positions itself toward solving the suffering of developers seeking correct declarative stack navigators.

Respond doesn't mess it around. It's of the *strong opinion* that the user's intent isn't for a new entry to appear on the stack, and perhaps erase all future entries, but rather to just increment/decrement the index. So it does so.

In this case, it takes the user back, which just so happens to be the direction the user was eager to move toward.

There's one major problem: the browser the vast majority of users use, Chrome (desktop and mobile), handles these pops asynchronously. Therefore if any changes (such as redirects) happen before current pops complete, the browser and Respond will become out of sync. 

To remedy this, Respond implements a cue, which guarantees subsequent history changes don't occur until the previous one (that Respond was expecting) is complete. The cue performs re-tries in short intervals of half an animation frame, 9 milliseconds.

So part of making this all work is addressing a few peculiar edge cases, such as the one being discussed here, where one press of the browser back button actually leads to 2 pops backward. We're able to detect such things, while not getting out of sync.

Other such edge cases include simply the redirect in the first place, even if not back to an adjacent entry. The behavior is also slightly different if these movements are triggered by dispatches of Respond's `back` and `next` action creators. 

The thing to keep in mind with all this is that a lot of these issues come from the fact that route transitions are not instantaneous, but rather async pipelines where authentication can asynchronously be addressed, and where it's expected that the route terminates if not validated or if the route redirects to somewhere else such as a login page.

Another such thing is route blocking. Sometimes you want to block a user from leaving a page such as an incomplete checkout (`beforeLeave`). Sometimes you want to block a user from entering a page such as a page that requires special authentication (`beforeEnter`). And both are places you may want to do async work. 

> And believe us, *these are things you want to do*. We know--these are THE MOST requested features from [redux-first-router](https://github.com/faceyspacey/redux-first-router) which became the basis of what you're reviewing today. 

> It's basically a world first to have these pristine sequential asynchronous pipelines to work with, given the control the browser's pop handling exerts. *It's a world first that Respond has finally overcome it.*

So the fact that route transitions are not necessarily instantaneous means users can try to do things in the middle of ongoing transitions. 

The browser back/next buttons are especially complex. 

UI-triggered actions, on the other hand, are a lot easier to handle since we don't need to normalize and undo URL changes we had no upfront control over.

In general, Respond has to jump through a lot of hoops to track when routes are short-circuited via redirects or returning `false` to stop a transition dead in its tracks. Then there's also users tapping the same button/link over and over again. It should be known that Respond blocks all "double dispatches." Don't expect to see the devtools stacked up with the same redundant actions over and over again, and the same thunks fetching the same data over and over again. But do expect newly triggered routes to cancel ongoing transitions that have yet to commit--the later one takes precedence.

We're still just scratching the surface. Respond has solutions for myriad other scenarios:

- multiple redirects within transitions (aka nested pipelines)
- nested pathless routes + anonymous thunks dispatching redirects
- 

The list goes on. The point is Respond makes important most solutions want to leave up to you in userland. By considering all these things, and having a few opinions about them, and through having an overall purpose as a driving light, Respond is able to connect the dots across disparate behaviors and capabilities to create a cohesive whole. 

For the most part, you don't have to think about this stuff. But for curious/advanced developers of integrity, this doc should increase your comfort levels--**yes, Respond lets you build apps as if the address bar, the sorely lacking `history` API, and browser are no longer in your way. Transparently as if they're non-existent.**



## Multiple + Nested Navigators

Last but not least, we said we'd address the issue of multiple and nested navigators (both stack navigators and other "navigators" like a tab bar).

Let's examine first of all examine how a single `StackNavigator` could look:
