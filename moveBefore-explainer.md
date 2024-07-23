# State-preserving move: Explainer

## Overview
DOM State-preseving move is a new primitive that fixes a long-standing web constraint: since moving
an element in the DOM means removing and re-inserting it, a lot of its state gets reset. The most
apparent effect of this is that when elements that contain iframes are moved around the DOM, the
iframes themselves get reloaded.

## The pain point
This is a pain point that have brought to existence pretty heavy library that try to mitigate it,
e.g. [morphdom](https://github.com/patrick-steele-idem/morphdom) that tries its best to move elements
"around" a stateful element to avoid removing it.

## What state gets reset
We've identified a short list of features that represent state that gets reset when an element is
removed and re-inserted:

    1. IFrames get reloaded
    1. A focused element loses its focus
    1. Text selection is cleared
    1. Fullscreen is existed
    1. A popover is closed
    1. A modal dialog ceases to be modal
    1. CSS animations & transitions are reset
    1. Pointer/touch events are cancelled



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