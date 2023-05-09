# Semantic Delegate

## Authors:

- Alice Boxhall (so far)

## Participate
- https://github.com/WICG/aom/issues/

<!--
## Table of Contents [if the explainer is longer than one printed page]

[You can generate a Table of Contents for markdown documents using a tool like [doctoc](https://github.com/thlorenz/doctoc).] 

-->

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This proposal aims to add a capability for a shadow roots to delegate one of its descendants to act in place of its host in the context of certain APIs.

## Motivating Use Cases 

Authors using shadow roots often want to be able to "wrap" an interactive element in order to take advantage of built-in semantics and behaviour.
This might look something like this:

```html
<fancy-input>
  #shadowRoot
  | <input>
  | <span>Fancy!</span>
  /#shadowRoot
</fancy-input>
```

> Note: I'm using a typographic convention to represent shadow roots
starting with `#shadowRoot`, using vertical bars `|` to indicate content which is within a shadow root, and ending with `/#shadowRoot` to indicate that
following content is outside the shadow root.
>
> These typographic elements shouldn't be taken literally as HTML content; 
I just find it easier to follow than either 
writing out imperative code to construct a shadow tree,
or using declarative Shadow DOM `<template>`s.

In the case above, the `<input>` within the shadow root is 
the critical piece of content which implements the "input-ness" 
of the `<fancy-input>`. 

However, this creates a problem:

```html
<label for="fancy">Your name:</label>
<fancy-input id="fancy" aria-describedby="hint">
  #shadowRoot
  | <input>
  | <span>Fancy!</span>
  /#shadowRoot
</fancy-input>
<span id="hint">This can be any name you would like to be addressed as.</span>
```

The `<label>` should really apply to the `<input>`, 
which is a focusable and labelable element,
and should have the string "Your name:" as its accessible name.
The `<input>`, in turn, should have the text contents of the `hint` span
as its accessible description. 
However, because of the encapsulation provided by the shadow root,
the `<input>` is not able to be refer to or be referred to by
elements outside the shadow root.

<!--
## Non-goals

[If there are "adjacent" goals which may appear to be in scope but aren't,
enumerate them here. This section may be fleshed out as your design progresses and you encounter necessary technical and other trade-offs.]

## User research

[If any user research has been conducted to inform the design choices presented
discuss the process and findings. 
We strongly encourage that API designers consider conducting user research to
verify that their designs meet user needs and iterate on them,
though we understand this is not always feasible.]

-->

## Semantic delegate API

The concept of the semantic delegate API is simply that
the shadow root should have a way to make one of its descendants
"stand in" for the shadow host. 
The actual API may take a number of forms, 
depending on experimentation. 
The naming in particular is subject to change.

With that in mind, one form the API might take is
a method or property on the shadow root 
which would allow the component author to specify 
one of the shadow root's descendents as the shadow root's semantic delegate:

```js
class FancyInput extends HTMLElement {
  constructor() {
    super();

    if (this.shadowRoot !== null)
      return;

    this.attachShadow({ mode: "open" });
    const input = document.createElement("input");
    this.shadowRoot.appendChild(input);

    // Mark the <input> as the semantic delegate for the shadow root
    this.shadowRoot.semanticDelegateElement = input;

    // Add whatever other exciting content is necessary
    this.addFanciness();
  }

  addFanciness() { ... }
}

customElements.define("fancy-input", fancy-input);
```

With this implementation of `<fancy-input>`, 
a page author could now use IDREF APIs with `<fancy-input>`
the same way they would with a plain `<input>`:

```html
<label for="fancy">Your name:</label>
<fancy-input id="fancy" aria-describedby="hint">
  #shadowRoot
  | <input>
  | <span>Fancy!</span>
  /#shadowRoot
</fancy-input>
<span id="hint">This can be any name you would like to be addressed as.</span>
```

This should have a declarative shadow DOM option as well;
perhaps something like:

```html
<label for="fancy">Your name:</label>
<fancy-input id="fancy" aria-describedby="hint">
  <template shadowrootmode="open" shadowrootsemanticdelegate="actualinput">
    <input id="actualinput">
    <span>Fancy!</span>
  </template>
</fancy-input>
<span id="hint">This can be any name you would like to be addressed as.</span>
```

## Open questions

### What should be in scope vs. out of scope for semantic delegation?

In the examples above, we assume `<label>` is affected by `semanticDelegate`.

However, this raises questions about what should reasonably be included
in the API's scope.

**Clearly out of scope**
- Styling shouldn't be affected: selectors which match the host shouldn't cause styles to be applied to the semantic delegate.
- your name^H^H^H^H addition here!

**Maybe?**
- `<label for>`
- `<label>` wrapped
- Form participation? Arguably, this is a type of implicit relationship like
  `<label>` wrapped?
- Some other things?

**Clearly in scope**
- ARIA!

### What should the syntax be?

The syntax outlined above is really just a strawman. It could be anything.

**Attribute-like**

Something like the examples above.

One benefit of this is that it lends itself easily to a declarative syntax.

**An shorthand to ARIA reflection/delegation**

If only ARIA is determined to be in scope, 
this could be a special case of the[ Cross-root ARIA delegation](https://github.com/leobalter/cross-root-aria-delegation/blob/main/explainer.md)
and [Cross-root ARIA reflection](https://github.com/Westbrook/cross-root-aria-reflection/blob/main/cross-root-aria-reflection.md) APIs.

For example:

```html
<custom-label id="foo">
  <template shadowroot="open"
            shadowrootreflectsariaattributes="all"
            shadowrootdelegatesariaattributes="all">
    <label reflectedariaattributes="all"
           delegatedariaattributes="all">
      Hello world
    </label>
  </template>
</custom-label>
 
<custom-input aria-labelledby="foo">
  <template shadowroot="open"
            shadowrootreflectsariaattributes="all"
            shadowrootdelegatesariaattributes="all">
    <input reflectedariaattributes="all"
           delegatedariaattributes="all">
  </template>
</custom-input>
```

That's a bit wordy for now, but we might be able to whittle those APIs down
such that it becomes more manageable.

**Method-like**

I'm not sure why we'd do this, but it could be a method on `shadowRoot`
rather than an attribute.


<!--
## Detailed design discussion

### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

-->

## References & acknowledgements

This proposal takes heavy inspiration from,
and may end up being simply a part of,
the [Cross-root ARIA delegation](https://github.com/leobalter/cross-root-aria-delegation/blob/main/explainer.md)
and [Cross-root ARIA reflection](https://github.com/Westbrook/cross-root-aria-reflection/blob/main/cross-root-aria-reflection.md) APIs.
As such, it derives from the work of
Leo Balter, Manuel Rego Casasnovas and Westbook Johnson on those APIs.

There is also some example code lifted from Nolan Lawson's excellent blog post,
[Shadow DOM and accessibility: the trouble with ARIA](https://nolanlawson.com/2022/11/28/shadow-dom-and-accessibility-the-trouble-with-aria/).
 