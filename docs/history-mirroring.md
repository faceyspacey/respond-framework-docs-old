# History Mirroring

Respond is the **first general purpose routing solution to successfully sync the browser's** ***hidden*** **history stack to one your app can see**. Until now, apps didn't bother tracking this stack because they thought they couldn't accurately sync it. 

As a result, apps could not know where visitors *have been*, only where they *are*. *"Web apps"* have been closer to *"web pages"*. Experiences such as wizards and stack navigators have not been fully functional. It's why you see horizontally animated navigators almost primarily on native, and few on web. Those days are now over.

Let's take a deep dive into the long-standing problem Respond has solved.


## The Problem

For those who haven't followed this problem space, basically the browser doesn't let you know what URLs you visited (even on a website you manage), when clearly it *itself* knows. For example, if you visited the following pages:

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
history = {
  entries: [
    ['https://www.respondframework.com', {}],
    ['[https://www.respondframework.com/docs', {}],
    ['https://www.respondframework.com/tutorial', { stateFoo: 'bar' }],
  ],
  index: 2,
  length: 3
}
```

Unfortunately, all the browser gives is information about the current URL and the `length` for the number of entries in that hidden array:

```js
window.history = {
  length: 3, // This isn't the length of only entries on your site, but for the whole tab! It's useless!
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

The `history` object has a few other methods, but they all pretty much amount to the same as pushing another URL. You certainly can't rewrite the whole history stack. More on that shortly.


> HONORABLE MENTION: in the past, solutions for stack navigators have tried to maintain history state in the URL itself, for example: `/tutorial/step-1/step-2/step-3`, but this failed for many reasons: *browser back/next buttons are impossible to sync (users can pop right off your own site), SEO is sacrificed since multiple URLs exist for the same page, URLs are messy and hard to remember, state for each entry is non-existent, etc.*



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

> As commented above, `history.length` includes entries from other sites in the current tab, and therefore can't be used to detect if you will `back` off the site.

So in actuality all we have is *push* and *replace*. 

What any wizard or stack navigator needs though is 3 capabilities:

1. **the ability to `reset` the entire stack**
2. **to `jump` the user to any index of the stack**
3. **to change (aka `set`) any entry on the stack**


### 1. Reset

Imagine you could dispatch a command/action that completely rewrites the entry stack:

```js
const index = 2
const direction = -1
const entries = [
  '/tutorial/step-1',
  '/tutorial/step-2',
  '/tutorial/step-3',
  '/tutorial/step-4',
]

actions.reset(entries, index, direction)
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

Respond makes precisely this possible. To pull this off, Respond changes the URLs as fast as it can, creating an almost imperceptible change in the address bar. *Yea, we went there!*

Let's look at a real life use case--resetting a checkout stack navigator after a purchase is complete:


#### Use Case:

```js
routes: {
  SUBMIT_CHECKOUT: ({ actions }, { payload }) => {
    const { success, reason } = api.post('checkout', payload)

    return success
      ? actions.reset(entries)
      : actions.flashMessage('Checkout submission failed because ' + reason)
  }
}
```

#### Additional `stack` Formats

`reset` also accepts entries in several forms:

```js
const entries =  [
  '/tutorial/step-1',                               // path string as above                    
  ['/tutorial/step-1', { some: 'state' } ],         // with a state object
  ['/tutorial/step-1', { some: 'state' }, '12345'], // with a user-provided unique identifier iey
  { type: 'TUTORIAL', params: { page: 'step-1' } }  // as an action object
]
```

### 2. Jump

In Respond, we renamed `go` to `jump` and imbued it with some additional capabilities:

```js
actions.jump(-2)                                                      // jump to the 0 index
actions.jump('12345')                                                 // jump to entry by its unique key
actions.jump(-2, { params: { foo: 'bar' }, state: { baz: 'boss' } })  // jump and change the URL + state
actions.jump(0, {}, true)                                             // jump by index
actions.jump(0, {}, true, 1)                                          // fake the direction in state
actions.jump('12345', {}, true)                                       // jump to entry with matching key
```

And for good measure Respond offers 2 shortcuts:

```js
actions.back() // actions.jump(-1)
actions.next() // actions.jump(1)
```

> `back` and `next` also accept an action-shaped object as an argument similar to `jump`

Why the rename? Because if you're moving one entry, the smart thing to do is use `back` or `next` to make clear the intent. Therefore, if you were ever to use `go`, it's to `jump` :)

In addition, our `jump` really is a big *jump* since you can also change its state, all the URL parts and even the action type. The 2nd argument accepts all the usual suspects: `params`, `query`, `hash`, `basename`, `state`. Leaving out the `type` is customary, but you could also provide it to jump to a route that was not there before.

All 3 actions will never bounce a visitor from your site. They're safe-guarded from that by throwing an exception. Which is an indicator you should use Respond's built-in `canJump` selector:

```js
const BackButton = (props, { canJump }, { back, step1 }) => (
  <button onClick={canJump(-1) ? back() : step1()}>
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

In summary, Respond detects when you're going to a previously visited page, but only when it's directly adjacent to the current entry. Because what else could you be doing? They share the exact same URL. They are the exact same page. You were just there! Your intent is to return to where you just were. `jump` is automatically used in these circumstances. This a powerful, automatic, yet opinionated and logical feature unique to Respond.

> And yes, you guessed it, Respond aims to solve the suffering of developers seeking correct declarative stack navigators. More on that below.



### 3. Set

Last but not least, you also need the ability to *set* different aspects on a history entry, *without actually re-visiting it.* `set` is like `jump` except the user stays where they are. It's most commonly used on the current entry:

```js
actions.set({ state: { flag: 'something' }, query: { rapper: 'Big Pun' } })
```

It allows you to set changes that skip the async route pipeline and efficiently go straight to the reducer + address bar. 

Here's how you set entries other than the current one:

```js
actions.set({ state: { flag: 'something' } }, -2)       // by delta
actions.set({ state: { flag: 'something' } }, '12345')  // by unique entry key
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

Respond follows this path to the bitter end. Because we control every aspect of routing, we are able to do things smaller libraries can't do. We're able to know where users went, from *start to finish,* as well as where they are in on ongoing transition. 

The challenge here is that during *non-instant* async transitions, it's very easy to lose track of how the browser sees the stack. Redirects and blocked route transitions throw a major wrench in the system. Rapid usage of the browser's back/next buttons and pipeline cancellations make matters worse. The biggest culprit however is that chrome doesn't perform *all* transitions synchronously!

Then there's also the challenge that sometimes `sessionStorage` isn't availbale, such as when browsers are in "incognito mode." Of course we have a solution for that as well: fallback to storing all info on each and every history state entry. More on that shortly.

And when browser's don't support history at all, we fallback to an in-memory solution that keeps the same URL the entire time. Browsers have advanced far enough that we have chosen to do this instead of a hash based solution like we previously had with [redux-first-router](https://github.com/faceyspacey/redux-first-router).

To give you confidence that Respond has you covered, let's examine the solutions to the challenges one by one:

## Session Storage

`sessionStorage` is what allows Respond to remember the history stack when visitors click links to other sites and return. Unlike `localStorage`, `sessionStorage` isn't remembered on subsequent visits once the given browser tab is closed. It's short-term storage just for a given tab while it's open. It's a perfect match for our goal of remembering just the current tab's stack of visited URLs.

This is a foundation we can build upon. From here, we need to make sure to never get out of sync. If we always stay in sync, we can just write to session storage the above info, which by the way looks closer to this:

```js
{
  index: 1,
  entries: [
    ['https://www.respondframework.com', { stateFoo: 'bar' }, '12345'],
    ['https://www.respondframework.com/tutorial/step-1', {}, '567378']
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

**Hey, we're missing an entry!** Yes, but it doesn't matter because, when returning from an external site, we had just the state we needed at the *tail* of the stack, and now the Respond app is running based on this location state. Respond won't continue to look into `window.history.state` on each history change. It only does so on the initial page returned to. 

Therefore, we can take what's correctly been put in state on the `/tutorial` page and overwrite what `/docs` has, and perhaps it will become handy if the user bounces from `/docs`. For example, we can now rewrite it as:


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

Lastly, Respond will of course recognize what `index` you returned on and remove subsequent entries to mirror how the browser sees the stack.

We won't get into all the heuristics, but the head of the stack has similar behavior handling.


## Worst Case Scenario

So far we have covered what most developers think of when it comes to mirroring the hidden history stack. Now let's cover what is the **bulk of the work in actuality.**

The root of the challenge can be broken down into 4 hurdles:

1. **Excessive Popping:** browser back/next buttons can change routes at will, and they can be tapped rapidly
2. **Automatic Jumping:** Respond automatically converts push/replace actions into jumps when re-visiting adjacent URLs, as that's the preferable user experience
3. **Queue Transitions for Chrome:** chrome doesn't push new entries synchronously!! ([this is a bug](https://bugs.chromium.org/p/chromium/issues/detail?id=775391)) Therefore we need to queue them.
4. **Request to Commit:** route transitions can be long and asynchronous, and there can be changes to transitions midway (redirects + blocking)



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
      // previous entries omitted for brevity
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
  from: { // route redirected from
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
  prev: { // previous route
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
}
```

The solutions to these hurdles rests on 4 pillars:


### 1. Undo Excessive Popping

The first hurdle Respond must deal with is obviously that the user should now be on, say, *news.ycombinator.com* because he/she tapped the back button in excess of 7 times.

Fortunately, Respond doesn't allow this. 

Respond's pop handler recognizes when the middleware pipeline is **"busy" as a result of a previous pop**. It then immediately **jumps one entry in the opposite direction to undo** this second pop. 

It's as if the pop never happened. *It's as if all 8+ pops, except the first one one, never happened.*

In effect, Respond normalizes all browser back/next button usage to **not honor** overzealous back/next button pressing **until one completes.** One at a time please!

> Respond even knows if the user quickly switches the opposite direction (i.e. tapped the *next* button).


### 2. Automatic Jumping

With that out of the way, let's discuss a topic you should be aware from above: **it's desirable to us for Respond to automatically jump instead of push (or replace) on occasions.** 

So at this point our entries are:

```js
[
  // previous entries ommitted
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
  // previous entries ommitted
  '/login',
  '/dashboard',
  '/login'
],
index: 7
```

or replacing so they become: 

```js
entries: [
  // previous entries ommitted
  '/login',
  '/login',
  '/'
],
index: 6
```

Respond is smart enough to know the real intent was to go back, **which means we need to jump back by 2 entries:**

```js
entries: [
  // previous entries ommitted
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

- If any changes (such as blocked routes or redirects) happen before current pops complete, the browser + Respond will get **out of sync**, and from there all hell would break loose. We don't want that.

To remedy this, Respond implements a **queue**, which guarantees subsequent history changes don't occur until the previous one is complete. The queue performs re-tries in short intervals of half an animation frame, 9 milliseconds.

Now we're able to insure no history changes happen that Respond doesn't know about. And since our middleware pipeline is built from the ground up around being asyncronous, it's no problem to *await* those extra 9 milliseconds, whereas with traditional routing solutions, this alone would pose a major problem requiring working arounds.


### 4. Requests to Commit

Respond's middleware essentially treats new dispatches as *requests* to push the given URL. The URL isn't changed until the middleware pipeline commits it. This is an *inversion of control* over the normal way route transitions are made (immediately when imperatively triggered).

Respond **normalizes** requests from the browser buttons so the middleware pipline can perceive them identically to actions triggered by UI events. In both cases, they are uncommitted *requests*. 

It then sends the `request` through a synchronous-feeling middleware pipeline that is easy to reason about. *This is the crown jewel we are shooting for.*

Here *redirecting* and *blocking* is trivial, provided we mirror the browser's history stack as we go.

The double jump back may seem irregular, but as you can see from the options above, it's actually the best and natural user experience. Such scenarios present a high likelihood of getting out of sync. There are other scenarios. This is just one of several. Without describing them all, **if you don't have each and every one covered, the whole system breaks down.** History loses sync.

So building route transitions around *requests* is less of a "solution" to our nightmare scenario, and more of our *primary goal* that we must uphold. It's where the redirect seamlessly happened after all. But by upholding it its linear nature, we also give ourselves a stable environment to move and control the history track underneath it. 

The request pipeline and the fully managed history stack work hand in hand to provide a stable linear code execution pipeline so you don't have to worry about it.


### Summary

If nothing else is accomplished from reading this article, understand that Respond takes it very seriously guaranteeing your app sees the same state as what your browser has in C++ somewhere. Respond doesn't just take you 80% of the way, and tell you to get out of the boat and swim to shore like most routers do. Respond considers all possible user experience heuristics like the *login, go back* scenario, which admittedly is a very rare scenario most apps leave as product holes, hoping users dont encounter them. 

Because of all that hard work, you can depend on the simplicity of route-dependent data requirements and fx. Hurray!







## Stack Navigators

Last but not least, we said we'd address the issue of Stack Navigators, especially when you have multiple or nested ones. In fact, Stack Navigators is less of an issue, and more of a the crowning achievement that proves we did everything right. While not seen often on web, we believe that with Respond they will become commonplace.

When it comes to components, Respond keeps a minimal surface area, thereby allowing you to draw on the standard ecosystem of React components. However, this is so important (and one closely tied to routing) that after `<Link />` and `<Route />` components, the next one we plan to implement is a `<StackNavigator />`. 

We haven't built it yet, but lets take a deep dive into how we *can build* on Respond's' routing foundation to easily achieve such things. Consider below as a *sneak peak* of what's to come.

To understand the challenges of multiple and nested navigators, let's first examine a single animated stack navigator. All it requires is 3 props:

- `entries`
- `index`
- `direction`

```js
const MyStack = (props, { location, components }, actions) => {
  const { entries, index, direction } = location

  return (
    <StackNavigator entries={entries} index={index} direction={direction}>
      {entry => {
        const Component = findComponent(entry, components)
        return <Component {...entry} />
      }}
    </StackNavigator>
  )
}

const findComponent = (entry, components) => {
  switch(entry.type) {
    case 'STEP1':
      return components.OrderForm || components.Loading
    case 'STEP2':
      return components.PaymentDetails || components.Loading
    case 'STEP3':
      return components.Confirm || components.Loading
    case 'STEP4':
      return components.ThankYou || components.Loading
    default:
      return components.NotFound
  }
}
```

Now let's look at one of the components corresponding to one of the steps:

```js
const Confirm = (props, state, actions) => ({
  <div>
    <OrderDetails />
    <Button onClick={actions.submitCheckout()}>PLACE ORDER</Button>
  </div>
})
```

Notice how there is no special props passed to the `Confirm` component. For example, `Confirm` isn't passed `props.navigation.navigate('ThankYou')` as in libraries like *react-navigation*. This is a good thing, as the goal is always *less API.* That's less for you to remember, learn, etc. 

The standard Redux-inspired action dispatching Respond is built on is all you need. `StackNavigator` will be smart enough to know what to do when it sees the `index` has changed and a new entry is pushed on to the `entries` array. And of course the sliding animation will be based on the intelligent `direction` state Respond maintains.

Here's what a complete `checkoutModule` for the above might look like:

```js
const checkoutModule = createApp({
  components: {
    MyStack,
    OrderForm,
    PaymentDetails,
    Confirm,
    ThankYou,
    NotFound
  },
  routes: {
    STEP1: '/step-1', // thunks related to steps omitted
    STEP2: '/step-2',
    STEP3: '/step-3',
    STEP4: '/step-4',

    // pathless route:
    SUBMIT_CHECKOUT: ({ state }) => { 
      const { success, reason } = api.post('checkout', state.order)

      return success
        ? actions.step4()
        : actions.flashMessage('Checkout submission failed because ' + reason)
    }
  },
  reducers: {
    order
  }
})
```

To really understand the gloriousness of Respond's upcoming `StackNavigator` requires at least being familiar with the APIs of other stack navigators like [react-navigation](https://reactnavigation.org). Nevertheless, the takeaway is simply:

**A fully loaded `StackNavigator` can be minimally and declaratively configured if correctness is guaranteed for what's contained in `entries`, `index`, and `direction`.**

To paint the picture, let's look at the advanced use case of resetting: 

```js
SUBMIT_CHECKOUT: ({ state }) => { 
  const { success, reason } = api.post('checkout', state.order)

  return success
    ? actions.step4()
    : reason === 'order invalid'
      ? actions.reset(['/home', '/step-1'])             // index 1 will automatically be inferred
      : actions.reset(['/home', '/step-1', '/step-2'])  // payment details issue
}
```

Respond is able to automatically determine that you want the sliding animation to be going backward. If you want to override it, you can set it:

```js
  actions.reset(['/home', '/step-1'], 1, 1) // the second 1 sets a forward direction
```

And it can handle complex scenarios where perhaps the index stays the same, and a completely different array of entries is used. Basically the `direction` prop--*which you control and has some sensible defaults for various scenarios*--dictates the sliding animation. And we will never see more than one scene slide by with Respond, as you sometimes do when other navigators are reset. Each change to the location state will at most result in one sliding animation.

Performance is addressed by holding in memory hidden renderings for adjacent views, based on the entry action they are derived from. What you can display is completely dynamic based on how you choose to transform the entry. 

So the one-to-one matching with the URL makes this stuff quite trivial, but how can we achieve the same thing for nested navigators? Can they no longer simply transform actions/entries into component views?? Do we need to dream up a `props.navigation.navigate` style API like *react-navigation*?

### What if there was a way to populate the `StackNavigator` with just the entries from the history stack that its concerned with?

Picture in additon to the above checkout, our mobile site also has a sidebar with a `StackNavigator`. The sidebar has the pages we're all too familiar with:

- Account
- Settings
- Payment 
- Help
- Etc

What happens if you're on `/step-2` and open the sidebar? Or click a link that takes you directly to the `Payment` page to view your default payment settings. Of course we want it to be as easy as just dispatching these actions:

```js
actions.settings()
actions.payment()
```

Here's what our entries array would look like in the latter case:

```js
['/home', '/step-1', '/step-2', '/payment']
```

And let's say we edit our default payment method:

```js
['/home', '/step-1', '/step-2', '/payment', '/confirm-payment-change']
```

And let's say we have `/confirm-payment-change` wired to redirect you back to your checkout if you're in the middle of one:

```js
['/home', '/step-1', '/step-2', '/payment', '/confirm-payment-change', '/step-3']
```


What we now have is entries corresponding to 2 StackNavigators within our global history stack. 

How do we reconcile the 2 navigators? What happens if you tap `back` within one of the given navigators? What happens to the other navigator? What happens to both the navigators when you tap the browser `back` button? 

How do we handle advanced scenarios like resetting one of the navigators? We certainly have to make sure to leave entries corresponding to the other navigator untouched. 

Most importantly, and in the first place, how do we *filter* a StackNavigator to just some of the entries? 

For now let's just assume that we used 2 modules, each with their own namespace, to create the routes used in the 2 navigators. That allows us to filter the entries like this:

```js
const SidebarStack = (props, { location, components }, actions) => {
  const entries = location.entries.filter(e => e.namespace === 'sidebar')
  const index = ???
  const direction = ???

  return (
    <StackNavigator entries={entries} index={index} direction={direction}>
      {entry => {
        const Component = findComponent(entry, components)
        return <Component {...entry} />
      }}
    </StackNavigator>
  )
}
```

But what about the `index` and `direction`? 

Let's start with the `index`.

It turns out the `index` can be derived by this algorithm:

- take the current global index and the entry it corresponds to
- if the entry corresponds to one filtered by the StackNavigator, you have a match
- if it *does not*, walk backward on the stack until you find the first entry that corresponds to the given StackNavigator
- in ***either case***, take the matched entry, and search the filtered entries for the first entry with a matching URL, and it's `index` is it

There is one caveat, the same entry can appear multiple times in the filtered set, so just knowing its URL and using it to find its index in an array isn't enough. In reality, the simplest algorithm for index discovery is *best performed while filtering the entries*. 

Also, StackNavigators don't need to be passed `entries`, `index` and `direction` because they can get all this information from Respond `context`, just like you can with the 2nd `state` argument passed to your components. 

Therefore our `StackNavigator` component can look just like this:

```js
<StackNavigator filter={entry => entry.namespace === 'checkout'}>
  {(entry) => {
    const Component = findComponent(entry, components)
    return <Component {...entry} />
  }}
</StackNavigator>
```

***That's it!***


### Implementation

But this intentionally lengthy article isn't just about getting to know Respond's API surface. Here our goal is to get down to the nuts and bolts of things, and learn the *why* and the *how* of everything. 

Let's take a look at the core aspects of an implementation:

```js
const StackNavigator = (props, state, actions) => {
  const { filter, children: render } = props
  const { entries, index } = filterEntries(filter, state.location) // tuck our algorithm away in here

  const { direction } = state.location

  const currentEntry = entries[index]
  const previousEntry = entries[index - direction] // find adjacent entry to slide away

  const currentComponent = render(currentEntry)
  const previousComponent = render(previousEntry)

  return renderSlidingAnimation(currentComponent, previousComponent, direction) // omitted
}
```

And now the algorithm for combined filtering and index discovery:

```js
const filterEntries = (filter, location) => {
  const { entries, index } = location

  return entries.reduce((acc, entry, i) => {
    if(filter(entry)) {
      acc.entries.push(entry)
    }

    // as you can see algorithmically its simplest and more performant (requires the least
    // number of passes) to discern the index at the same time as filtering the entries
    if (index === i) {
      acc.index = acc.entries.length - 1 // the length at this iteration is the hint we seek
    }

    return acc
  }, { entries: [], index: 0 })
}
```

With this simple setup we've accomplished being able to use the global history stack as the **single source of truth** for multiple stack navigators!

That means, you can use the browser back/next buttons, and each StackNavigator will know what to do. They will derive the correct index automatically, rather than require you to specify it. And all this can happen under the hood, since Respond components have direct access (using React `context`) to global `location` state.


### But what if you want an easy `back` api for an individual StackNavigator?

It turns out, we can use additional render props/arguments for that too:

```js
<StackNavigator filter={entry => entry.namespace === 'checkout'}>
  {(entry, { back, canJump }) => {
    const Component = findComponent(entry, components)

    return (
      <>
        {canJump(-1) && <button onClick={back()}>back</button>}
        <Component {...entry} />
      </>
    )
  }}
</StackNavigator>
```

*implementation:*

```js
const StackNavigator = (props, state, actions) => {
  // ...
  let { canJump } = modifySelectors(state)
  let { back, next } = modifyActions(actions)
  const currentComponent = render(currentEntry, { back, next, canJump, etc })
  // ...
}
```

We won't get into the exact implementation here, but the idea is the `StackNavigator` component can intervene and adjust the precise `back`, `canJump`, etc functions provided, modifying the core global ones.

Going back to our original history stack, the result of jumping `back` within one navigator *over entries from another navigator* might look like this:


- **BEFORE:** ['/home', '/step-1', '/step-2', '/payment', '/confirm-payment-change', ***'/step-3'***], index: 5

- **AFTER:** ['/home', '/step-1', ***'/step-2'***, '/payment', '/confirm-payment-change', '/step-3'], index: 2

So that means all the modified `back` action must do is `jump` by 3. Incredible! It really is as simple as that.

As far as `next`, the algorithm is even simpler: `push` is used, erasing all subsequent entries on all stacks, similar to the built-in global history stack behavior. The exception is if the entry pushed is already the `next` entry, which again is identical behavior to the global history stack.

The thing to note about the `next` action is that it's less important than having a generic `back` action you can rely on. The reason is that for `next` you always must know where you're going, but with `back` providing that information is essentially redundant. Stack navigators are like a linked list, and the "link" is only required in one direction. Most importantly, it's great for prototyping to reliably put a `<BackButton />` and be done. 


### Does back, canJump actions passed to components know they are in a StackNavigator?

As mentioned, Respond components use React `context` to gain access to global routing information. Similar to Respond module namespacing implementation, Respond can make the necessary modifications if a component is nested within a StackNavigator. 

Using the `back` action and `canJump` selector therefore works the same everywhere:

```js
const BackButton = (props, { canJump }, { back }) => canJump(-1) ? (
  <button onClick={back()}>
    back
  </button>
) : null
```


### What about reseting just a specific navigator?

Algorithmically that might look something like this:

```js
const checkoutEntries = entries.filter(e => e.namespace === 'checkout')
const sidebarEntries = entries.filter(e => e.namespace === 'sidebar')

// closing sidebar strategy:
actions.reset(['/settings', ...checkoutEntries]) 

// staying on sidebar strategy:
actions.reset([...checkoutEntries, '/settings', '/payment']) 
``` 

What's the difference between the 2 strategies? 

Well, that depends where you currently are. If you're on a checkout entry, then the first strategy will reset the other navigator behind the scenes, and the 2nd one will reset the current navigator, performing the appropriate sliding animation. That's pretty much all there is to it. 

You can do this manually yourself in, for example, a route callback. Or, if you're within a StackNavigator, you can get help:

```js
<StackNavigator filter={entry => entry.namespace === 'checkout'}>
  {(entry, { back, reset }) => {
  // ... you know how the story ends
```

Here, `reset` will adjust just the current StackNavigator, automatically choosing one of the above 2 strategies. Or you could force them respectively like this:

```js
reset(entries, false) // force push entries to the head
reset(entries, true)  // force push entries to the tail
```

In both those strategies, `reset` will set the index to that of the last entry in the array. However, you can manually set the `index` and `direction` with additional arguments, just like in the regular `reset` method.

You can also pass an index to splice the entries into:

```js
reset(entries, 7)
```


### Resets Without Sliding Animations

Occasionally you want to prevent the sliding animation on reset, and just immediately show the desired scene:

```js
reset(entries, false, entries.length - 0, 0)
```

By passing `0` for the `direction` no animation is used.


### Secondary Entries

It's your choice whether a given navigator scene has secondary entries that occur within it. If such is the case, you need a mechanism to further filter which entries trigger new scenes, and which do not. A `key` prop passed to the component returned from the render prop does the trick:

```js
<StackNavigator filter={entry => entry.namespace === 'checkout'}>
  {(entry, { routes }) => {
    const Component = findComponent(entry, components)
    const key = routes[entry.type].scene
    return <Component {...entry} key={key} />
  }}
</StackNavigator>
```

Keys can be generated any way you want. The overall idea is you need a way to group which routes belong on what scene. You can easily do that by attaching this information to your routes:

```js
routes: {
    STEP1: {
      path: '/step-1',
      scene: 'step1'
    },
    FOO: {                    // non-navigator-changing route
      path: '/step-1/foo',
      scene: 'step1'          // see, it has the same scene key as STEP1
    },
    STEP2: {
      path: '/step-2',
      scene: 'step2'
    },
    BAR: {                    // non-navigator-changing route
      path: '/step-2/foo',
      scene: 'step2'          // see, it has the same scene key as STEP2
    },
    etc
```

 > If you can come up with an automatic strategy that doesn't require configuration information, go for it.



## Direct Visits & `initialEntries`

We've now seen the sort of advanced capabilities (e.g. stack navigators) that can be built upon an information-complete foundation. We'd like to take a second to fill in one final gap you may not be aware of: 

- **direct visits to scenes within Stack navigators**

If you were to visit, say, step 3, in the above checkout, you'd need to insure the previous scenes are in the entries array so `back` functionality would work as expected, right?

In many scenarios you'd also need to insure various data was fetched via the middleware pipeline and route callbacks, right? 

What you need is this capability:

```js
route.initialEntries = [
  '/checkout/step-1',
  '/checkout/step-2',
  '/checkout/step-3'
]
```

as well as in dynamic function form:

```js
route.initialEntries = (request, action) => {
  return [
    '/checkout/step-1',
    '/checkout/step-2',
    '/checkout/step-3'
  ]
}
```

What this does is force the browser to visit these routes as quickly as possible, replaying them. It's done on initial *load* of your app only, calling all middleware and callbacks along the way. 

> That means different routes can have an `initialEntries`, and it's only called when the given route is the route the app loads on.

This guarantees the necessary state is produced for `/checkout/step-3` to behave correctly.

You can't do such things without the capability to rewrite the history stack. Forget about it!

### DRY ("Don't Repeeat Yourself")

Lastly, it's important to note that you can also provide `initialEntries` *once* on a parent route that groups a bunch of child routes:

```js
routes: {
  PARENT: {
    routes: {
      STEP1: '/checkout/step-1',
      STEP2: '/checkout/step-2',
      STEP3: '/checkout/step-3'
    },
    initialEntries: [
      '/checkout/step-1',
      '/checkout/step-2',
      '/checkout/step-3'
    ]
  }
}
```

And by doing so, the initial entries will automatically replay the routes up until the  directly visited route. That way you don't have to provide `initialEntries` multiple times for each and every route in the wizard or stack navigator. If you so choose, you can override what a parent provides by specifying a different value for `initialEntries` on the child.



## Hoisted Data Dependencies + Effects


By now it should be clear why Respond invested so much in not just history mirroring but also history manipulation capabilities: 

- **It's a required pre-requisite for hoisting data dependencies and side-effects to the route level** ***in all scenarios.***
- Effects grouped and orchestrated together linearly in a single place is the correct most natural way they should be addressed if possible, not randomly discovered in component trees.
- High quality apps and user experiences must avoid shortcuts at all costs. From an app development experience, it's trivial and natural to do things like specify an `initialEntries` route option. The browser shouldn't present roadblocks to such things *just working.*

Features like `initialEntries` and the automatic replay on direct visits may seem like a stretch (it sure took us a lot of work!). And there are ways to get around it, for example the aforementioned less desirable redirect strategy. When regular React apps have stack navigators, they "solve it" by avoiding making changes to the URL all together! 

However we determined it necessary to insure the route level approach wasn't a leaky abstraction, even if most apps don't require features like StackNavigators. Our prediction, however, is that will change in the future.

Development is all about the "right tool for the job." And the right tool for effects is something that linearly groups them together. We're essentially taking effects spread randomly throughout your component tree and putting them back in order, the way they're supposed to be.

The component level stuff that rose to prominence for a while was a stop-gap because all these things weren't in place. You shouldn't be forced to think of your app at a *microscopic* level (component level fetching), when the *macro* big picture of your app clearly designates that a given URL or area of your app has certain dependencies.

There are certain things that still make a lot of sense being developed with the "low level" React API. For example, the Facebook chat widget served on their desktop site for years. That's a great use case for Hooks and Suspense. 

Respond isn't at odds with React. It's a perfect combination. The Facebook chat widget  could be naturally built the Respond way with "sticky components" and routes. But there may be good reasons doing it with traditional component level strategies makes the most sense for your app. And you can do such things directly within a shell provided by Respond. The few occasions that's a right fit feel so much better with Respond providing overall flow and conventions.

Generally speaking, Respond adds a *macro* way for your app to be modular **in addition** to *micro* component modularity.


In conclusion, history mirroring + manipulation is a required capability in order for URL-driven data dependencies to be widely realizable. Without it, Respond can't make the claims it makes in regards to the widespread feasability of hoisted data deps.

> OUR CLAIM: The majority of features we build are better served hoisting data deps. Even rare features like chat widgets can easily be built with Respond routes and modules. The chance that what you're building can't be naturally built at the route level is slim to none. The Respond architecture is now a required tool in your toolbox.


And if you're wondering why much of the React community has given up on hoisted deps until now, it's that the reliability was leaky at best. And most commonly, it was non-existent. There answer seems to have been: "forget about the URL, just keep it the same," as stack navigators and similar kinds of components have demonstrated.

After all, history mirroring and manipulation had never been generally achieved until now. *There's no package you can get on NPM to do just the history control part of Respond.* 

The overall outcome is that a proven and reliable approach from the server-side world--due to its linear execution--has been unable to flourish in a client-centric world. Let's face it, sneaking activities that make sense together into pockets of component tree branches isn't a better developer experience. It's a tradeoff made to achieve component modularity. However, if you can do both, depending on what the situation calls for, you're going to be in the DX sweet spot. 

What we have found is that 75-100% of data-dependent features are part of the *main application flow* where Respond is the clear winner. The the rest are exceptions like the chat widget where Hooks + Suspense is a fine option for data-fetching, depending on your needs. But again, Respond's "sticky components" + routes also works beautifully for such component modules.


## Final Thoughts

This obviously was an important article in terms of *Respond theory.* It's not necessarily important *in practice*. Novice developers can disregard it if it doesn't make complete sense. More advanced developers can stride more confidently, knowing what's going on under the hood and why we chose the approach we chose. **Long live linear execution!**


**Meditation:**

*your routes embody horizontal segways between separate component tree renderings*

*you're no longer shackled by your host environment, the browser*

*you can treat commits to the address bar no differently than commits to state*

*you have the space before and after commits to seamessly do what you like, workarounds not required*

*and if you need you can rewrite history*

*you finally have a foundation to build things that depend on the browser's hidden history state.*

