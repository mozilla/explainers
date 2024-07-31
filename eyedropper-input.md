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

The EyeDropper API only supports sRGB. &lt;input type=color> also currently only supports sRGB, but there’s an [issue to add wide-gamut support](https://github.com/whatwg/html/issues/3400).

The EyeDropper API has no declarative button built-in.


## Proposed solution

See [Consider extending &lt;input type=color> instead](https://github.com/WICG/eyedropper-api/issues/35).


## Examples

Minimal HTML example:

```html
<input type=color eyedropper onchange="setPaintColor(this.value)">
```

Existing EyeDropper API code would continue to work (assuming no renaming), but the constructor would create an &lt;input type=color eyedropper> element.

```js
// Create an EyeDropper object (<input type=color eyedropper>)
let eyeDropper = new EyeDropper();

// Enter eyedropper mode
let icon = document.getElementbyId("eyeDropperIcon")
icon.addEventListener('click', e => {
	eyeDropper.open()
	.then(colorSelectionResult => {
    	// returns hex color value (#RRGGBB) of the selected pixel
    	console.log(colorSelectionResult.sRGBHex);
	})
	.catch(error => {
    	// handle the user choosing to exit eyedropper mode without a selection
	});
});
```


## Caveats, shortcomings, and other drawbacks of design choices, both current and any prior iterations

The “open” method is a bit generic for HTMLInputElement. Maybe it can be renamed?

&lt;input> is mainly event-based, and the current EyeDropper API is promise-based. Is it reasonable to do both?

The EyeDropper API [uses [SecureContext]](https://github.com/WICG/eyedropper-api/issues/15), which might be hard with &lt;input>. Is it needed?


## Draft specification

To be written.


## Incubation and/or standardization destination

Incubation could be in WICG/eyedropper-api, standardization in WHATWG HTML.
