# Advanced Route Config


## `route.initialEntries: Array<Entry>`

This allows the client to replicate those entries seamlessly (by erasing current entries and replaying the contents of the array). This addresses hard refreshes when a visitor has already visited many URLs on your app and has as a result already stacked up many entries recorded in the browser's `history`.

Another use case for this (regardless of whether you're doing SSR) is if for example if the user lands on step 3 of your checkout--`route.defaultEntries` allow you to fill in steps 1-3, so back functionality works seamlessly. 

## `route.middleware`
## `route.redirect`
## `route.inherit`
## `route.parseSearch`
## `route.stringifyQuery`