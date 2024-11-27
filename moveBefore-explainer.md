# State-preserving move: Explainer

## Overview
DOM State-preseving move is a new primitive that fixes a long-standing web constraint: since moving
an element in the DOM means removing and re-inserting it, a lot of its state gets reset. The most
apparent effect of this is that when elements that contain iframes are moved around the DOM, the
iframes themselves get reloaded.

### The pain point
This is a pain point that have brought to existence pretty heavy library that try to mitigate it,
e.g. [morphdom](https://github.com/patrick-steele-idem/morphdom) that tries its best to move elements
"around" a stateful element to avoid removing it. Other libraries like [htmx](https://htmx.org/) have
elaborate workarounds for this issue.

### What state gets reset
We've identified a short list of features that represent state that gets reset when an element is
removed and re-inserted:

  1. IFrames get reloaded
  1. A focused element loses its focus
  1. Selection is cleared
  1. Top level UI is closed (fullscreen, popover, modal)
  1. CSS animations & transitions are reset
  1. Pointer/touch events are cancelled

## The proposed solution

### The API

The proposed new API is a new DOM function: `Node.prototype.moveBefore`. It's (almost) a drop-in replacement for
[`Node.prototype.insertBefore`](https://dom.spec.whatwg.org/#dom-node-insertbefore), and behaves in the 
exact same way, except for the following:

1. The element state defined above does not get reset. IFrames stay loaded, focused elements stay focused
   (as long as they didn't move to an inert tree), CSS transitions continue or get triggered, etc.
2. The author gets reflection of this in web components: a new optional `movedCallback`, which is invoked
   instead of `disconnectedCallback` and `connectedCallback` when an element that has it declared is moved
   in a state-preserving manner.

See discussion at https://github.com/whatwg/dom/issues/1255.

### Constraints

Both the node and new container need to be either [connected](https://dom.spec.whatwg.org/#connected) or disconnected, and part of the same document.
When these constraints are not met, `moveBefore` throws an exception. That is because, at the moment, there is no design that allows us to move nodes
across documents or have them change their connected state without side effects like script execution, which would in some cases be widely inconsistent
with "moving".

### A few specifics

#### IFrames
When an iframe is moved, it does not reload, and does not fire events.

#### Focus
When a focused element is moved, it generally stays focused. In the next [focus fixup](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model%3Afocusing-steps), it might lose focus and fire blur events, if for example it was moved into a hidden or inert subtree.

### Selection
Selection is currently not preserved. This can be addressed in future version. At the moment, moving is constrained to "intrinsic" state of the node, and not to state that relates to other nodes, like ranges.


## Considered alternatives

### An iframe attribute

e.g. a `preserve` attribute. this felt like the wrong layer to implement such a feature, and the moving primitive is anyway designed to do much more than iframe state preserving.

### Changing the default behavior of existing DOM methods

This was thoroughly considered and attempted.
Once constraint that allowed us to perform an "atomic" move is the fact that the API works on one element at a time.
Modern APIs, like [`before`](https://developer.mozilla.org/en-US/docs/Web/API/Element/before) or [`append`](https://developer.mozilla.org/en-US/docs/Web/API/Element/append), accept multiple elements, some of which can be moved and some of which cannot.
This complicates the API a lot, and we haven't found a satisfactory solution that would feel safe as a drop-in replacement for these.

This leaves us with [`insertBefore`](https://developer.mozilla.org/en-US/docs/Web/API/Node/insertBefore), [`appendChild`](https://developer.mozilla.org/en-US/docs/Web/API/Node/appendChild) and [`replaceChild`](https://developer.mozilla.org/en-US/docs/Web/API/Node/replaceChild).
Those have been around for a long time, which brings a web-compatibility risk. In addition, applying a move semantic to 3 APIs and not to the rest would create confusing inconsistency in an API where consistency is key.

### Falling back to `insertBefore` when moving is not possible

This is a [controversial topic](https://github.com/whatwg/dom/issues/1255).

A request that repeats a lot from web developers is for `moveBefore` to be a drop-in replacement for `insertBefore`, that works under the same conditions.
However, this would make `moveBefore` inconsistent, as it would move the node without side-effects under some conditions, and incur side-effects, some of which might be major like reloading iframes or running scripts, under other conditions.

It is possible for callers to turn `moveBefore` into a drop-in replacement for `insertBefore` by checking for [`isConnected`](https://developer.mozilla.org/en-US/docs/Web/API/Node/isConnected) or by catching the move-specific exceptions,
and most early adopters of this API from the library/framework space are likely to use it in this way to incorporate them into their existing code.

However, it is not embedded into the initial version API for deliberate reasons - `moveBefore` predictably moves the node without side-effects. This is something developers can count on, given the conditions.
Developers are encouraged to think about "moving" as a bespoke DOM operation, rather than as a "better insert". By doing so, developers can craft their use of the DOM APIs in a way that *always* moves when it can,
rather than "try to move but fall back", inevitably resulting in a user experience where iframes are *sometimes* reloaded and focus is *sometimes* lost.

In other words, the DOM API goes to a pretty low level. Having a primitive that *just* moves and fails if it can't is the prudent first baby step for this new functionality.

## Possible future enhancements

### `appendChild` and `replaceChild` versions

We could have convenience functions that move the node to the end, or remove a child and move the node in its place. This should be a somewhat simple addition.

### A version that falls back to insertion

A method like `moveOrInsertBefore` can be a drop-in replacement for `insertBefore` that moves when it can and insert when it can't.
As mentioned before, we should tackle this later on, when we understand how useful `moveBefore` is as a drop-in replacement for `insertBefore`, vs. using it as a new primitive at a higher level.
This should come after some time has passed, when the adoption of this API goes a bit beyond incorporating it into existing code that was originally tuned to a world where atomic moves were impossible.

### Batch moves

To incorporate move operations into batch functions like [`append`](https://developer.mozilla.org/en-US/docs/Web/API/Element/append), we'd have to design the semantic of how this should behave when trying to append some elements that are
movable and some that are not, and whether this changes the order of removal/insertion effects. It might be possible to add a version that throws if *any* of the nodes is not movable, however it's unclear how useful that is.
In addition, we can envision some sort of "transaction" model where several operations take place, and the effect is either moves or insertions, based on the initial and final state of the nodes, disregarding intermediate state.

Also, given such mechanism, it might be possible to think of preserving tree state such as selection ranges.
To conclude, moving multiple elements at the same time is orders of magnitude more complex than moving a single element, which is complex by itself, hence it is deferred to a future API.

## [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

> 01.  What information does this feature expose,
>      and for what purposes?

It does not "expose" any new information, but rather allows skipping 

> 02.  Do features in your specification expose the minimum amount of information
>      necessary to implement the intended functionality?

Yes, they don't expose anything more than the new function.

> 03.  Do the features in your specification expose personal information,
>      personally-identifiable information (PII), or information derived from
>      either?

No

> 04.  How do the features in your specification deal with sensitive information?
> 05.  Does data exposed by your specification carry related but distinct

>      information that may not be obvious to users?

No

> 06.  Do the features in your specification introduce state
>      that persists across browsing sessions?

No

> 07.  Do the features in your specification expose information about the
>      underlying platform to origins?

No

> 08.  Does this specification allow an origin to send data to the underlying
>      platform?

No

> 09.  Do features in this specification enable access to device sensors?

No

> 10.  Do features in this specification enable new script execution/loading
>      mechanisms?

No

> 11.  Do features in this specification allow an origin to access other devices?

No

> 12.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?

No

> 13.  What temporary identifiers do the features in this specification create or
>      expose to the web?

No

> 14.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?

It's not applicable to this API

> 15.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?

N/a

> 16.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?

The DOM specification [does](https://dom.spec.whatwg.org/#security-and-privacy), but it's not
really applicable.

> 17.  Do features in your specification enable origins to downgrade default
>      security protections?

No

> 18.  What happens when a document that uses your feature is kept alive in BFCache
>      (instead of getting destroyed) after navigation, and potentially gets reused
>      on future navigations back to the document?

Nothing special, this API only works when used on an active document.

> 19.  What happens when a document that uses your feature gets disconnected?

The feature is built in such a way that a document cannot be disconnected in the middle of using
the feature. When used on disconnected documents, this feature falls back to existing behavior.

> 20.  Does your feature allow sites to learn about the users use of assistive technology?

No

> 21.  What should this questionnaire have asked?

Nothing in particular.
