# History Mirroring

Respond is the **first general purpose routing solution to exist on planet earth to successfully sync the browser's hidden history stack to the stack your app keeps**. Until now, apps didn't bother tracking such a stack because they knew they couldn't keep it in sync, and therefore maintain its accuracy. 

Apps could not know where the visitors have been, only where they *are*. As a result *"apps"* have been closer to *"pages"* and *"websites"* than they could be. It's why you see horizontally animated navigators on native so much more often. 

> Until now, websites have been the opposite of fertile ground for anything powered by stack navigators.


## The Problem

For those who haven't followed this problem space, basically the browser doesn't let you know what URLs you visited (even on a website you manage), even though clearly it itself knows. For example, if you visited:

```js
[
  'https://www.reddit.com/r/reactjs',
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
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

It's understandable that browsers won't let your app know what other sites your visitors visited, but it **won't even let you know about itself!**


Imagine the things you could do if you always knew exactly where a user was along the track of history entries:

```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
],
index: 2,
length: 3
```

Let's make sure you have a feel for this by navigating around a bit--imagine you tap `back` once:

```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
],
index: 1,
length: 3
```

And `back` again:

```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs',
  'https://www.respondframework.com/tutorial',
],
index: 0,
length: 3
```

Imagine you go from the homepage to a different next page:


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs/installation',
],
index: 1,
length: 2
```

Now entries on the current index and onward have been erased. The array has shrunk in length, and the latest entry has been replaced with a new URL.

Let's do it again:


```js
entries: [
  'https://www.respondframework.com',
  'https://www.respondframework.com/docs/installation',
  'https://www.respondframework.com/docs/getting-started'
],
index: 2,
length: 3
```

Now imagine you could dispatch a command that completely rewrites the entry stack (both in terms of how your app perceive it and how the browser does):

```js
actions.reset([
  'https://www.respondframework.com/tutorial',
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2',
  'https://www.respondframework.com/tutorial/step-3',
], 1)
```

That will obviously result in this stack:

```js
entries: [
  'https://www.respondframework.com/tutorial',
  'https://www.respondframework.com/tutorial/step-1',
  'https://www.respondframework.com/tutorial/step-2',
  'https://www.respondframework.com/tutorial/step-3',
],
index: 1,
length: 4
```

Keep in mind, you're not just changing how your app perceives the stack, giving it a false impression; *you're also changing the URLs that the browser's address bar just saw!* Now you're in complete control of the browser history stack.

> Of course to pull this off, Respond changes the URLs as fast as it can, creating a barely perceptible change in the address bar. On mobile, it's typically invisible since the address bar is above the fold.

This is the sort of information and capabilities that would make trivial--or at least straightforward--a great number of professional use cases:

- Stack Navigators
- Navigator Resets
- Reliable Back/Next buttons within your app (that don't need to be aware of where they're going)
- Jumping to previous pages
- Setting state on previous pages
- Rewriting other URLs in the stack
- Multiple Navigators, Nested Navigators (more on this later)
- Default Entries for direct visits (e.g. to /checkout/step-4)
- **Amazing control of checkouts and wizards**
- **Reliable back/next animations on mobile**

## Manual Syncing

Attempts have been made in the past to manually sync an *in-memory* data structure (i.e. some variables within your website) to what it thinks the browser contains, but they've all given up on what needs to be done. 

Respond follows this path to the bitter end. Because we control every aspect of routing, we are able to do things partial solutions and isolated libraries can't do. We're able to know where users went *start to finish.* 

The challenge here is that during non-instant async transitions, it's very easy to lose track of how the browser sees the stack. Redirects and blocked route transitions throw a major wrench in the system. The primary culprit however is rapid usage of the browser's back/next buttons. 

That's the meat of the work and there are some tough browser quirks we had to overcome.

The only remaining challenging after that is when browsers don't support `sessionStorage`. And we have a solution for that as well: fallback to storing all information on each and every history state entry. More on that shortly.

And when browser's don't support history, we fallback to an in-memory solution that keeps the same URL the entire time. We have chosen to do this instead of a hash based solution like [redux-first-router](https://github.com/faceyspacey/redux-first-router).


## Session Storage

`sessionStorage` is what allows Respond to remember the history stack when visitors click links to other sites and return. Unlike `localStorage`, `sessionStorage` isn't remembered on subsequent visits once the given browser tab is closed. This is a perfect impedence match for our goal of remembering just the current tab's stack of visited URLs.

This is a foundation we can build upon. From here, we need to make sure to never get out of sync. If we always stay in sync, we can just write to session storage the above info, which by the way look closer to this:

```js
{
  index: 1,
  entries: [
    ['https://www.respondframework.com', { stateFoo: 'bar' }, 12345],
    ['https://www.respondframework.com/tutorial/step-1', {}, 12345]
  ]
}
```

This doc won't go into the details of insuring we never get tripped, but just take our word for it when we say the hardest part of Respond's implementation is insuring we don't blunder.


## History Storage Fallback

For browser's that don't support `sessionStorage`, such as browsers in *incognito* mode, we had to think outside the box, and we think our solution is quite ingenius.

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

On `/docs` however we have this stored:

```js
window.history.state = {
  index: 1,
  entries: [
    ['https://www.respondframework.com', {}, 12345],
    ['https://www.respondframework.com/docs', {}, 45632],
  ]
}
```

**Hey, we're missing an entry!** Nope. It doesn't matter because, when returning from an external site, we had just the state we needed at the *tail* of the stack. 

> The same is true for *head* of the stack.

Therefore if the visitor hits back again, we'll override the state stored for `/docs`:


```js
window.history.state = {
  index: 1, // notice the index is 1 for /docs
  entries: [
    ['https://www.respondframework.com', {}, 12345],
    ['https://www.respondframework.com/docs', {}, 45632],
    ['https://www.respondframework.com/tutorial', {}, 71956],
  ]
}
```

And if the visitor clicks another external link and then returns, `/docs` will once again have all the necessary info. 

**But now there's a dangling entry for `/tutorial`!** Solving that problem is easy: Respond automatically truncates all entries after the current index on return, just as is standard behavior when backing to a previous URL and then clicking to another URL within your own site.

We won't get into all the heuristics, but the head of the stack has similar behavior handling.



## Redirects, Blocking, Async Transitions and Rapid Back/Next Tapping


Ok, we said we weren't going to explain the complexities here, but we decided we'll give you a taste in order to give you confidence that Respond provides a proper foundation to build the advanced functionality it's work so hard to enable.

