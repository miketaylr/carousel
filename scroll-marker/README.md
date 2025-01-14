# Scroll-marker elements and pseudo-elements

## Problem

Authors need to be able to easily create a set of focusable markers for all of their items,
or pages of items when combined with automatic fragmentation.

For individual items, an author *can* do this manually,
though it requires writing extra elements
which need to be kept up to date with the items to which they scroll.
Script also needs to be used to get the desired scrolling behavior.

For dynamically content-sized pages, this can only currently be done with script which generates DOM.
By having a way to automatically generate markers,
many more advanced UI patterns can be solved in CSS.

### Requirements

Scroll markers require the combination of several behaviors:

1. They should scroll to the target on activation,
2. only one of the scroll markers for a given scroller should be active (and focusable) at a time (see [roving tab-index](https://developer.mozilla.org/en-US/docs/Web/Accessibility/Keyboard-navigable_JavaScript_widgets#technique_1_roving_tabindex)),
3. arrow keys should cycle between the other markers,
4. the correct one is automatically activated as a result of scrolling, and
5. The active marker should be stylable.

## Proposals

The following proposes how scroll markers can be achieved via elements and via pseudo-elements.

### Elements

There are two different proposals for how we might define scroll marker elements.
The first tries to modify invokers and focusgroups as little as possible,
with the hope that we could fully explain the requirements via those proposals.
The second wraps up the requirements into a new invoker-like target.

#### Invoker action and focusgroup invoke action

We add a new built-in `invoke-action` (see [invokers](https://open-ui.org/components/invokers.explainer/)) `scrollTo`. When invoked, the `invokeTarget` will be scrolled to within its ancestor scrolling container. E.g.

```html
<button invoketarget="my-section" invokeaction="scrollTo">Scroll to section</button>
...
<section id="my-section">
  This will be scrolled into view when you click the button
</section>
```

Invoker actions are only [invoked](https://open-ui.org/components/invokers.explainer/#terms) on explicit activation,
and interest actions are shown [interest](https://open-ui.org/components/interest-invokers.explainer/#terms) on focus *or* hover.
For scroll markers, we want the action to be taken only when the selected target changes, which occurs on focus, but not on hover.
This is very similar to an expressed intent to invoke the target.

As such, we propose adding the `invoke` keyword to the `focusgroup` attribute to allow invoking the `invokeaction` on focusgroup focus changes. E.g.

```html
<style>
  #toc {
    position: sticky;
    top: 0;
  }
</style>
<ul class="toc" focusgroup="invoke">
  <li><button tabindex="-1" invoketarget="section-1" invokeaction="scrollTo">Section 1</button></li>
  <li><button tabindex="-1" invoketarget="section-2" invokeaction="scrollTo">Section 2</button></li>
  <li><button tabindex="-1" invoketarget="section-3" invokeaction="scrollTo">Section 3</button></li>
  <li><button tabindex="-1" invoketarget="section-4" invokeaction="scrollTo">Section 4</button></li>
</ul>
<section id="section-1">...</section>
<section id="section-2">...</section>
<section id="section-3">...</section>
<section id="section-4">...</section>
```

Note that this example uses tabindex="-1" to apply the [roving tab index with a guaranteed tab stop](https://open-ui.org/components/focusgroup.explainer/#guaranteed-tab-stop) behavior from focusgroup.

This propsoal notably does not meet requirements 4 and 5 of scroll markers.

#### HTML element / attribute

It is not possible to explain all of the requirements with the existing [invokers](https://open-ui.org/components/invokers.explainer/) and [focusgroup](https://open-ui.org/components/focusgroup.explainer/) proposals.
In particular, the first section provides a means by which we could explain requirements 1-3, but not requirements 4 and 5.

It may make more sense to instead add a new `scrolltarget` attribute or scrollmarker element which provides 1, 2, 4, 5, and possibly 3 (though this could be left to focusgroup).

E.g. using an attribute:

```html
<div class=toc>
  <button scrolltarget="section-1">Section 1</button>
  <button scrolltarget="section-2">Section 2</button>
  <button scrolltarget="section-3">Section 3</button>
</div>
```

Or, using an element:

```html
<div class=toc>
  <scrollmarker target="section-1">Section 1</scrollmarker>
  <scrollmarker target="section-2">Section 2</scrollmarker>
  <scrollmarker target="section-3">Section 3</scrollmarker>
</div>
```

### Pseudo-elements

Using pseudo-elements is the *only* way to declaratively handle dynamic cases
where the number of elements generating markers is not known (e.g. based on [fragmentation](../fragmentation/)).

We create a `::scroll-markers` pseudo-element on [scroll containers](https://www.w3.org/TR/css-overflow-3/#scroll-container).
This pseudo-element will implicitly have `contain: size`,
and be positioned after the `:after` pseudo-element.

The `::scroll-marker` pseudo-element will create a focusable marker which when activated will scroll the element into view.
This pseudo-element will be flowed into the `::scroll-markers` pseudo-element of its containing scroll container.

```css
ul {
  overflow: auto;
}
ul::scroll-markers {
  display: flex;
  width: 100%;
  /* Reserve space for scroll markers */
  height: 40px;
  /* Allow scrolling if too many scroll markers are inserted. */
  overflow: auto;
}
li::scroll-marker {
  content: "#";
}
```

The created scroll markers implement the [tabs pattern](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/), in particular:
* Implicitly form a [focusgroup](https://open-ui.org/components/focusgroup.explainer/) with all other scroll markers for the same scroller.
  The currently active scroll marker will have a persistent class applied to it.
* Each marker has tab-index -1 ensuring that exactly one will be in the tab index per the [guaranteed tab stop](https://open-ui.org/components/focusgroup.explainer/#guaranteed-tab-stop) focusgroup behavior such that only the active marker is focusable. E.g. focus will move to the active scroll marker, past any other inactive markers.
* When focus is on a scroll marker, per the [focusgroup attribute](https://open-ui.org/components/focusgroup.explainer/#quickstart):
  * Left/up arrow moves focus to and activates the previous scroll marker.
  * Right/down arrow moves focus to and activates the next scroll marker.

In addition, these markers automatically respond to other scrolling operations.
When any scrolling operation takes place,
the first marker which is considered to be scrolled into view becomes active, but is not focused.

### The active marker

Within a particular scrolling container, a single scroll marker is determined to be active.
The active scroll marker is determined as:
* The one which scrolling into view requires the least scrolling to bring into view.
* In a tie, if one is an ancestor of the other, the ancestor is not considered,
* Of the remaining elements whose scroll distance to scroll into view is the same,
  the first in tree order is selected.

When asked to [run the scroll steps](https://drafts.csswg.org/cssom-view/#document-run-the-scroll-steps)
the active marker should be updated according to the eventual scroll location that the scroller will reach
based on the current scrolling operation.

#### Styling the active marker

The active marker is considered to be toggled to an on state and can be styled using the [:checked pseudo-class](https://developer.mozilla.org/en-US/docs/Web/CSS/:checked).

## Example

Typically, scroll markers will be used with [grid-flow](../grid-flow/) to create navigation points.

See an [example](https://flackr.github.io/carousel/examples/scroll-marker/) built on the polyfill.
