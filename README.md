# AdBlockPlus + "Mirage"
A generic method for preventing the detection of **element hiding**, a method of ad blocking.

![Adblock Plus + "Mirage" Demo](mirage-demo.gif)

## Overview
This approach works by intercepting all element style inspection functions and disabling the ad block styles before the function call and re-enabling them after.
This happens fast enough that the style changes are not visually seen (or repainted).
Caching of ad element styles allows this approach to be performant and not slow down the browsing experience.

## Resource Blocking
Resource blocking is another method of blocking ads.
It is preferred over, or in addition to, element hiding since it enhances privacy, reduces resource usage, and arguably increases safety.

Unfortunately it is not clear how to generically bypass script blocking detection.
JavaScript resource blocking can be detected pretty easily: For example a flag will not be set if the script is not executed.
Custom scripts can be used to counter this but they will be ad block detector and or site specific, and will most likely need to change over time.

Therefore it is suggested that all resource blocking is disabled when trying to bypass ad block detection with this method.

You can do that by adding these rules to whitelist all resources:
```
@@$genericblock
@@$script
```

Finally some ad block rules seem to rely primarily on the resource not being loaded for blocking.
You might see ads which have no element hiding rules for them.

## Building, Etc.
See [the original AdBlockPlus Readme](README.AdBlockPlus.md)

## Tested On
- [Fuck Ad Block](https://fuckadblock.sitexw.fr)
- [Forbes](https://www.forbes.com/forbes/welcome/)
- [WashingtonPost.com](https://www.washingtonpost.com/news/post-politics/wp/2017/08/14/trump-denounces-kkk-neo-nazis-as-justice-department-launches-civil-rights-probe-into-charlottesville-death/?hpid=hp_hp-top-table-main_pp-trump-125pm%3Ahomepage%2Fstory&utm_term=.2e7807bae717)

## Detailed Description
Specific element style related functions are intercepted so that they return the value they would have returned if the ad block style was not applied.
This is done by disabling ad block styles before the function call, evaluating the function, and reenabling the ad block styles.
The object prototype functions intercepted are HTMLElement.{offsetTop,offsetLeft,offsetWidth,offsetHeight,offsetParent}, Element.{clientWidth,clientHeight}, and CSSStyleDeclaration.getPropertyValue.

To keep things snappy, the element styles are cached and recomputed in batch.
This is necessary since toggling styles *twice* per function call becomes quite expensive; especially when some of these functions are called in time sensitive areas such as on `mousescroll` events.

The cache is invalidated when a DOM element is added, removed, or a style attribute is modified.
These events can alter what an element's computed style might be.

As another means of improving performance, the ad block style has been modified to trigger an animation event ([more details](https://stackoverflow.com/questions/6997826/alternative-to-domnodeinserted)).
The elements which trigger this event are added to the ad elements set.
This is much faster than calling `querySelectorAll` on DOM mutations.

Using this technique means `display: none` cannot be used to hide the ads since an animation event will not be triggered.
Instead `visibility: hidden` and `position: absolute` are used for element hiding.

## Limitations
Since ad element detection is being done through asynchronous events (`animationstart`), an ad element is not always detected due to a race condition.
An element can also have no events fired for it if the element is added and removed from the DOM too quickly. This has not been too big of an issue in the past but recently it has become more noticeable on certain sites.
This condition becomes much more likely when an ad element is added, inspected, and removed in rapid succession.

Alternative synchronous approaches such as determining if an element is an ad element during inspection causes a noticeable performance impact on complex pages, even with caching.
Worse still memory usage drastically increases due to the inclusion of the ad blocking selectors on each page/iframe needed for calls to `Element.match()`. It does however work reliably.

# Future Work
This approach to preventing the detection of element hiding is working well enough for a proof of concept.
Caching has helped significantly with performance.

There is much room for improvement here and below are some options worth exploring:

## Parallel DOM
A copy of the page's DOM is stored in the shadow root without any ad blocking styles applied.
The parallel DOM is kept in sync via monitoring DOM mutations in the main page and applying them to the shadow DOM.
A mapping of page ad elements to shadow ad elements will need to be stored.
When a page's ad element style is inspected the style of the mapped shadow element will be returned.

I have some concerns about memory usage but my hunch is that the DOM, ignoring resources, is pretty small, and resource references will be shared between the page and shadow DOM since they are on the same page.

This approach seems the most robust.

## Computing Style Changes Ourselves
Changes to DOM style are monitored and new styles parsed while ignoring the ad block styles.
If an element blocking method such as `visibility: none;` is used then element layout will not need to be recomputed.
This means that only the styles which affect the `visibility` state of the ad elements need to be parsed and stored.
When an ad element is inspected only the stored `visibility` state needs to be returned.

This approach seems the most difficult and does not allow collapsing of the ad space.
