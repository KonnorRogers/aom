# Semantic Delegate

## Authors:

- Alice Boxhall (so far)

## Participate
- https://github.com/WICG/aom/issues/

## Introduction

Many Web Components use a pattern I'd like to call a "semantic delegate" pattern,
where a built-in element (such as an `<input>`) is "wrapped" inside a shadow root,
and provides much of the basic functionality of the element.

This document explains what that pattern looks like in practice and
what challenges it causes for authors,
and proposes that we explore options to support this pattern via a
new API which would become part of the Web Components suite.

## The "Semantic Delegate" custom element authoring pattern

Authors using shadow roots often want to be able to "wrap" an interactive element
in order to take advantage of built-in semantics and behaviour.
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
> starting with `#shadowRoot`, using vertical bars `|` to indicate content
> which is within a shadow root, and ending with `/#shadowRoot` to indicate that
> following content is outside the shadow root.
>
> These typographic elements shouldn't be taken literally as HTML content;
> I just find it easier to follow than either
> writing out imperative code to construct a shadow tree,
> or using declarative Shadow DOM `<template>`s.

In the case above, the `<input>` within the shadow root is
the critical piece of content which implements the "input-ness"
of the `<fancy-input>`.
This follows a well-worn HTML authoring best practice of using built-in
HTML elements wherever possible, rather than trying to re-implement them.

Since this pattern implicitly uses the wrapped built-in (i.e. semantic) element
as a delegate to compose some primary functionality into the custom element,
I have been referring to it as the "semantic delegate" pattern.

### Some examples of custom elements from component libraries which use this pattern:

- [FAST `<text-field>`](https://github.com/microsoft/fast/blob/master/packages/web-components/fast-foundation/src/text-field/text-field.template.ts#L31) wraps an `<input>`
- [FAST `<disclosure>`](https://github.com/microsoft/fast/blob/master/packages/web-components/fast-foundation/src/disclosure/disclosure.template.ts#L10) wraps `<details>`/`<summary>`
- [FAST `<combobox>`](https://github.com/microsoft/fast/blob/master/packages/web-components/fast-foundation/src/combobox/combobox.template.ts#L26) which wraps an `<input>` (but seems to also secretly allow _decorating_ an `<input>`, which blows my mind)
- [Polymer `<paper-input>`](https://github.com/PolymerElements/paper-input/blob/master/paper-input.js#L171) - interestingly, this wraps an `<iron-input>` which _decorates_ an `<input>`
- [Spectrum `<sp-checkbox>`](https://opensource.adobe.com/spectrum-web-components/components/checkbox/) which wraps an `<input type="checkbox">`
- [Spectrum `<sp-action-menu>`](https://opensource.adobe.com/spectrum-web-components/components/action-menu/) which wraps a `<sp-action-button>`
- [Shoelace `<sl-input>`](https://github.com/shoelace-style/shoelace/blob/c31d4f5855a504a687642697c7e54971028f254b/src/components/input/input.component.ts#L461-L465) wraps an `<input>`
- [Shoelace `<sl-textarea>`](https://github.com/shoelace-style/shoelace/blob/c31d4f5855a504a687642697c7e54971028f254b/src/components/textarea/textarea.component.ts#L346) wraps a `<textarea>`
- [Shoelace `<sl-button>`](https://github.com/shoelace-style/shoelace/blob/c31d4f5855a504a687642697c7e54971028f254b/src/components/button/button.component.ts#L259-L264) wrap both `<a>` and `<button>` depending on if an `href` is supplied.

> Note: I use the term _"wrapping"_ to indicate a custom element which "bundles" an element inside its shadow root, so that an author using the custom element can simply use it, like `<fancy-input>`.
>
> Some custom elements use a _"decorating"_ pattern instead, where an author has to "pass in" one or more elements to be decorated/enhanced as part of the custom element's API. This is how [`<iron-input>`](https://github.com/PolymerElements/iron-input/blob/master/iron-input.js#L20) works.
>
> My usage of these terms isn't from any kind of agreed-upon standard; this is just my idiosyncratic terminology. If there _is_ agreed-upon terminology, I will gladly update my vocabulary and this doc!

## Problems caused by the Semantic Delegate pattern

### Cross-shadow IDREFs (and `label` wrapped)

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

Also, while not technically an IDREF association, using a `<label>` wrapped
around a `<fancy-input>` won't work either:

```html
<label>Your name:
<fancy-input id="fancy">
  #shadowRoot
  | <input>
  | <span>Fancy!</span>
  /#shadowRoot
</fancy-input>
</label>
```

### Form participation

```html
<form>
  <fancy-input>
    #shadowRoot
    | <input>
    | <span>Fancy!</span>
    /#shadowRoot
  </fancy-input>
</form>
```

Similarly, custom elements which use this pattern have to do extra work
to allow the wrapped `<input>` to participate in a `<form>`.
The [form-associated custom elements](https://html.spec.whatwg.org/dev/custom-elements.html#custom-elements-face-example)
APIs make this _possible_, but it is redundant work when the `<input>`
would automatically have its value associated with the `<form>`
if it wasn't inside a shadow root.

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

### What types of "semantic delegation" can reasonably work?

The API loosely proposed above specifies a single element within a shadow root
which acts as the "delegate" for the shadow host's behaviour.
Clearly, not all custom components' behaviour may reasonably be delegated to a
single element.

- What are some examples of complex custom components which contain _multiple_
  "delegate" elements? A date picker?

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

## Illustration of how a semantic delegate API would make custom element authors' lives easier

Ben Howell has put together a
[very thoroughly considered proposal](https://github.com/behowell/aom/blob/exportid-explainer/exportid-explainer.md)
for a mechanism for allowing elements' IDs to be "exported" and referred to from outside
of a shadow root, taking inspiration from [CSS part](https://drafts.csswg.org/css-shadow-parts/#part)
and [`exportparts`](https://drafts.csswg.org/css-shadow-parts/#element-attrdef-html-global-exportparts).

This proposal would make it technically possible for authors to set up the cross-shadow
IDREF associations which are currently impossible to express
(and, to be clear, I am _very_ strongly in support of continuing work on this proposal
and hopefully shipping something very much like it.)

However, for even a moderately complex component, the amount of ID exporting,
forwarding and aliasing necessary quickly gets
[dizzying](https://github.com/behowell/aom/blob/exportid-explainer/exportid-explainer.md#example-5-a-kitchen-sink-example-of-a-combobox):

```html
<label for="x-combobox-1::id(the-input)">Example combobox</label>
<x-combobox id="x-combobox-1">
  #shadowRoot
  | <x-input
  |   forwardids="the-input"
  |   useids="my-activedescendant: x-listbox-1::id(opt1),
  |           my-listbox: x-listbox-1::id(the-listbox)">
  |   #shadowRoot
  |   | <input
  |   |   role="combobox"
  |   |   id="the-input" exportid
  |   |   aria-controls=":host::id(my-listbox)"
  |   |   aria-activedescendant=":host::id(my-activedescendant)"
  |   |   aria-expanded="true"
  |   | />
  | </x-input>
  | <button aria-label="Open" aria-expanded="true">v</button>
  |
  | <x-listbox id="x-listbox-1">
  |   #shadowRoot
  |   | <div role="listbox" id="the-listbox" exportid>
  |   |   <div role="option" id="opt1" exportid>Option 1</div>
  |   |   <div role="option" id="opt2" exportid>Option 2</div>
  |   |   <div role="option" id="opt3" exportid>Option 3</div>
  |   | </div>
  | </x-listbox>
</x-combobox>
```

In order to follow (an earlier version of) this example,
I had to print it out and use highlighters to follow how the IDs are connected,
ending up with something like this:

![Screenshot of above code snippet, with colour highlighting to show how IDs are exported, forwarded, receieved, and finally used. For example, instances of the string "my-activedescendant" are all highlighted purple, while "opt1" is highlighted blue, to show how the aria-activedescendant attribute value on the wrapped <input> comes to refer to an element in a sibling shadow root. 7 colours are necessary to disambiguate all the IDs that are used.](images/combobox-syntax-1.png)

Conversely, with something like the API proposed above, it would be possible
to almost completely avoid using the `something::id(something-else)` syntax
and, in this example, to completely avoid needing to re-map IDs using `useids`:

```html
<label for="x-combobox-1">Example combobox</label>
<x-combobox id="x-combobox-1">
  #shadowRoot (semantic delegate -> "x-input")
  | <x-input
  |   id="x-input"
  |   aria-controls="x-listbox-1"
  |   aria-activedescendant="x-listbox-1::id(opt1)"
  |   aria-expanded="true"
  |   #shadowRoot (semantic delegate -> "the-input")
  |   | <input
  |   |   role="combobox"
  |   |   id="the-input"
  |   | />
  | </x-input>
  | <button aria-label="Open" aria-expanded="true">v</button>
  |
  | <x-listbox id="x-listbox-1">
  |   #shadowRoot (semantic delegate -> “the-listbox”)
  |   | <div role="listbox" id=“the-listbox”>
  |   |   <div role="option" id="opt1" exportid>Option 1</div>
  |   |   <div role="option" id="opt2" exportid>Option 2</div>
  |   |   <div role="option" id="opt3" exportid>Option 3</div>
  |   | </div>
  | </x-listbox>
</x-combobox>
```

![Screenshot of above code snippet, also with colour highlighting, showing how "opt1" is still exported but now used on the `<x-input>` instead, and how the delegation simplifies the rest of the ID references, so that only 3 colours are necessary because, for example, "x-combobox-1" delegates to "x-input" which delegates to "the-input", making them synonymous.](images/combobox-syntax-2.png)

## References & acknowledgements

This proposal takes heavy inspiration from,
and may end up being simply a part of,
the [Cross-root ARIA delegation](https://github.com/leobalter/cross-root-aria-delegation/blob/main/explainer.md)
and [Cross-root ARIA reflection](https://github.com/Westbrook/cross-root-aria-reflection/blob/main/cross-root-aria-reflection.md) APIs.
As such, it derives from the work of
Leo Balter, Manuel Rego Casasnovas and Westbook Johnson on those APIs.

There is also some example code lifted from Nolan Lawson's excellent blog post,
[Shadow DOM and accessibility: the trouble with ARIA](https://nolanlawson.com/2022/11/28/shadow-dom-and-accessibility-the-trouble-with-aria/).

