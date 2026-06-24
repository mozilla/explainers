# EyeDropper as &lt;input> explainer

Author: [Simon Pieters](https://github.com/zcorpan)

## User problems to be solved

Users want to be able to pick a color from the screen, e.g. in a painting or photo editor web app in order to use a color from the current canvas/photo or possibly from another application. Typically this is done with a button to activate “eyedropper mode” where the cursor becomes an eyedropper to pick a color. Existing web APIs don’t provide a way to read pixel data outside of &lt;canvas>.


## Methodology for approaching & evaluating solutions

Research the UX of existing eyedropper color selectors in native apps and web apps. Is it activated from a button, keyboard shortcut, etc? Is it closed when a color is picked, or stays open? How are non-sRGB colors handled? Can colors outside the viewport be picked? Are colors picked continuously while hovering, or only when clicking? Make sure the proposed solution is sufficiently rich to be adopted, but also doesn’t introduce privacy or security issues.

The proposed solution should be such that it does not duplicate effort when improving aspects of other color features, e.g. wide gamut for &lt;input type=color>. Ideally, the proposed solution will build on and work with existing color features in the platform.

The proposed solution should be easy to use and style declaratively without needing to implement a custom button with JS, and advance our [vision for the Declarative Web](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb).


## Prior features and proposals

[Proposal: &lt;input type=color> hint for eyedropper-only](https://github.com/whatwg/html/issues/5584) was filed in May 2020, and is essentially the same as the proposal in this explainer.

[The EyeDropper API proposal](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/EyeDropper/explainer.md) was created in August 2020, which was [moved to WICG](https://github.com/wicg/eyedropper-api/) in August 2021. The &lt;input type=color> proposal was closed without further discussion. [Chromium shipped EyeDropper API in M95](https://chromestatus.com/feature/6304275594477568).

[iam-medvedev/eyedropper-polyfill](https://github.com/iam-medvedev/eyedropper-polyfill) is a JS-based polyfill for the EyeDropper API.

[Building a wide-gamut colour picker](https://www.purplesquirrels.com.au/2023/12/building-a-wide-gamut-colour-picker/) is a JS-based color picker with P3 support (not an eyedropper).


## Flaws or limitations in existing features/proposals

The EyeDropper API only supports sRGB. `&lt;input type=color>` supports wide-gamut with [`colorspace=display-p3`](https://html.spec.whatwg.org/#attr-input-colorspace).

The EyeDropper API has no declarative button built-in.


## Proposed solution

See [Consider extending &lt;input type=color> instead](https://github.com/WICG/eyedropper-api/issues/35).

## Examples

Minimal HTML example:

```html
<input type=color eyedropper onchange="setPaintColor(this.value)">
```

JavaScript example:

```js
// Create an <input type=color eyedropper>
const eyeDropper = document.createElement('input');
eyeDropper.type = 'color';
eyeDropper.colorSpace = 'display-p3';
eyeDropper.eyeDropper = true;
eyeDropper.onchange = e => {
  console.log(e.target.value);
};

// Enter eyedropper mode
const icon = document.getElementbyId("eyeDropperIcon")
icon.addEventListener('click', e => {
	 eyeDropper.showPicker();
});
```


## Caveats, shortcomings, and other drawbacks of design choices, both current and any prior iterations

&lt;input> is mainly event-based, and the current EyeDropper API is promise-based.

A previous iteration of this proposal suggested specifying an `EyeDropper` constructor to create an `input` element with the appropriate attributes set, and an `open()` method, as a way to improve compatibility with existing web content. This has now been dropped for simplicity.

The EyeDropper API [uses [SecureContext]](https://github.com/WICG/eyedropper-api/issues/15), which might be hard with &lt;input>. The proposal above does not use SecureContext, but has other mitigations to prevent attacks.


## Draft specification

The `input` element would have a new boolean attribute `eyedropper`, which only applies when the `type` attribute is in the [Color](https://html.spec.whatwg.org/#color-state-%28type%3Dcolor%29) state. When specified, the element represents a color well control that opens an eye dropper.

For the purpose of the [show a picker, if applicable](https://html.spec.whatwg.org/#show-the-picker,-if-applicable) algorithm, the relevant user interface is an eye dropper, to select a color from a pixel on the screen, or within the browser, as appropriate for the platform and the user agent.

The eye dropper user interface has an explicit commit action and no intermediate manipulation. The end user's selection occurs only when the user explicitly selects a color using the eye dropper.

The user activation that caused the eye dropper to open must not itself select a color.

After the eye dropper is opened, the user agent must not allow a color to be selected until the eye dropper user interface has been visibly presented for at least 500ms.

The user agent must also ignore a color-selection activation that the platform would treat as part of the same multi-click sequence as the activation that opened the eye dropper.

> Note:
> This protects against click-jacking, where the user is tricked into double-clicking and unknowingly selects a color.

When the `eyedropper` attribute is present:

* The `colorspace` attribute applies.
* The `alpha` attribute does not apply (must be ignored and must not be specified).
* The `list` attribute does not apply (must be ignored and must not be specified).

The user agent must at most update the value once each time the eye dropper is opened.

> Note:
> Updating the value continuously while the user moves the eye dropper around, or as content changes underneath, could leak information.

While the eye dropper is open, device input events (e.g. mouse and keyboard events) must be suppressed.

> Note:
> This makes it harder for an attacker to know which pixel was selected, or to move elements in response to moving the eye dropper.

If, while the eye dropper is open, the element's `eyedropper` attribute is removed, the element's `type` attribute is no longer in the [Color](https://html.spec.whatwg.org/#color-state-%28type%3Dcolor%29) state, the element is no longer [mutable](https://html.spec.whatwg.org/#concept-fe-mutable), or the element's [node document](https://dom.spec.whatwg.org/#concept-node-document) is no longer [fully active](https://html.spec.whatwg.org/#fully-active), then the user agent must dismiss the eye dropper without selecting a color.

User agents must allow users to dismiss the eye dropper without selecting a color.

When the eye dropper is **dismissed without selecting a color** for an `input` element *element*, the user agent must [queue an element task](https://html.spec.whatwg.org/#queue-an-element-task) on the [user interaction task source](https://html.spec.whatwg.org/#user-interaction-task-source) given *element* to [fire an event](https://dom.spec.whatwg.org/#concept-event-fire) named `cancel` at *element*, with the `bubbles` attribute initialized to true.

If the user successfully selects a color using the eye dropper for an `input` element, then the [value](https://html.spec.whatwg.org/#concept-fe-value) must be set to the result of [serializing a color well control color](https://html.spec.whatwg.org/#serialize-a-color-well-control-color) given the element and the end user's selection.

For the purposes of [updating and serializing a color well control color](https://html.spec.whatwg.org/#serialize-a-color-well-control-color), an `input` element whose `eyedropper` attribute is specified must be treated as if its `alpha` attribute were not specified.

Whenever the `eyedropper` attribute is added or removed, if the element's `type` attribute is in the [Color](https://html.spec.whatwg.org/#color-state-%28type%3Dcolor%29) state, the user agent must run [update a color well control color](https://html.spec.whatwg.org/#update-a-color-well-control-color) for the element.

IDL:

```
partial interface HTMLInputElement {
  [CEReactions, Reflect] attribute boolean eyeDropper;
};
```

### Rendering

#### The input element as an eye dropper control

An `input` element whose `type` attribute is in the [Color](https://html.spec.whatwg.org/#color-state-%28type%3Dcolor%29) state and has an `eyedropper` attribute specified, is [expected](https://html.spec.whatwg.org/#expected) to depict a color well, which, when activated, provides the user with an eye dropper to allow selecting a color from the screen.

> Issue:
> What should the button contain? An "eye dropper" icon would be good, but also showing the selected color. Further, it should be possible to customize (at least with `appearance: base`).

## Incubation and/or standardization destination

Incubation could in a whatwg/html PR, standardization in WHATWG HTML.
