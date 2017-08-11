# AdBlockPlus + "Mirage"
A proof of concept generic method for blocking the detection of AdBlock.

## Overview
This approach works by intercepting all element style inspection functions and disabling the ad block styles before the function call and re-enabling them after.
This happens fast enough that the style changes are not visually seen (or repainted).

However this approach is rather resource intensive as it triggers two style recalculations on *every* intercepted function call of an ad element.
This can prove problematic where the intercepted functions are called in rapid succession such as the on scroll event, which has been observed on `www.washingtonpost.com`.

## Resource Blocking
Unfortunately it is not clear how to generically bypass script blocking detection. JavaScript resource blocking can be detected pretty easily; for example a flag will not be set if the script is not executed. Custom scripts can be used to counter this but they will be ad block detector and or site specific, and will most likely need to change over time.

Therefore it is suggested that all resource blocking is disabled when trying to bypass adblock detection with this method.

You can do that by adding these rules to whitelist all resources:
```
@@$genericblock
@@$script
```

Finally some adblock rules seem to rely primarily on the resource not being loaded for blocking. You might see ads which have no element hiding rules for them.

## Building, Etc.
See [the original AdBlockPlus Readme](README.AdBlockPlus.md)

## Tested On
- [Fuck Ad Block](https://fuckadblock.sitexw.fr)
- [d3xt3r: Anti Adblock](http://d3xt3r.com/anti-adblock?test)
- [Forbes.com](https://www.forbes.com/#7ff3a4692254)
- [WashingtonPost.com](https://www.washingtonpost.com/politics/trump-dictated-sons-misleading-statement-on-meeting-with-russian-lawyer/2017/07/31/04c94f96-73ae-11e7-8f39-eeb7d3a2d304_story.html?tid=sm_tw&utm_term=.80b19f510879)
- [ThePirateBay.org](https://thepiratebay.org/search/test/0/99/0)

## Detailed Description
Specific element style related functions are intercepted so that they return the value they would have returned if the ad block style was not applied.
This is done by disabling ad block styles before the function call, evaluating the function, and reenabling the ad block styles. The object prototype functions intercepted are
HTMLElement.{offsetTop,offsetLeft,offsetWidth,offsetHeight,offsetParent}, Element.{clientWidth,clientHeight}, and CSSStyleDeclaration.getPropertyValue.

A weak set of ad elements and a weak set of computed styles for those ad elements are stored.
The second set is necessary since the values it references are dynamically computed.

To keep things snappy the ad block style has been modified to trigger an animation event ([more details](https://stackoverflow.com/questions/6997826/alternative-to-domnodeinserted)).
The elements which trigger this event are added to the ad elements set.
This is much faster than calling `querySelectorAll` on DOM mutations.

Using this technique means `display: none` cannot be used to hide the ads since an animation event will not be triggered. Instead `visibility: hidden` and `position: absolute` are used for element hiding.


# Future Work
This biggest issue here is performance. The only two ways around this that I can see are 1) Caching DOM style values, or 2) Computing style changes instead of toggling the style sheet and forcing  a layout computation.

## Caching DOM Style
### Element Caching
The simplest approach is to cache the ad element's computed style when the ad block style is disabled.
However the cache will need to be invalidated when *any* style changes, or a DOM element which affects the layout is added or removed. This last constraint could be relaxed if ad detection is not looking at the ad element's position.

This might work pretty well if the DOM is not being modified very often.

### Parallel DOM
A copy of the page's DOM is stored in the shadow root without any ad blocking styles applied. The parallel DOM is kept in sync via monitoring DOM mutations in the main page and applying them to the shadow DOM. A mapping of page ad elements to shadow ad elements will need to be stored. When a page's ad element style is inspected the style of the mapped shadow element will be returned.

I have some concerns about memory usage but my hunch is that the DOM, ignoring resources, is pretty small, and resource references will be shared between the page and shadow DOM since they are on the same page.

This approach seems the most robust.

## Computing Style Changes Ourselves
Changes to DOM style are monitored and new styles parsed while ignoring the ad block styles. If an element blocking method such as `visibility: none;` is used then element layout will not need to be recomputed. This means that only the styles which affect the `visibility` state of the ad elements need to be parsed and stored. When an ad element is inspected only the stored `visibility` state needs to be returned.

This approach seems the most difficult.
