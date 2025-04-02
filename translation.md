# Translation API

Author: [Kagami Rosylight](https://github.com/saschanaz)

## User problems to be solved

Some web services (like social services e.g. Mastodon/Twitter or websites with reviews like Airbnb/Amazon) often provide a translation button (e.g. “Translate post”) for posts in languages different from the UI of the service. This frequently requires servers to communicate with paid third party services (e.g. in Twitter: “Translated from Spanish by Google”).

Many small services often find it difficult to pay extra maintenance fees for third party translation services, and thus are unable to provide translations of some of their content to users.

## Methodology for approaching & evaluating solutions

All major browsers provide the ability for users to translate a web page, with varying levels or degrees of support of languages from and to. We will explore solutions that allow websites to make use of such built-in browser translation capabilities to support translation buttons on pages or portions thereof (like specific posts or sections of a page).

We will prefer solutions that are more user-privacy respecting, like minimizing (ideally to zero) leaking any additional local browser configuration, e.g. whether the browser has downloaded a local translation model for a specific pair of languages or not.

### Out of scope: 

1. Exposing API to choose other third party translation services. That can be done from the browser side if needed.

2. Providing rich information about source text, e.g. providing dictionary definitions.

3. Non-text translation, deferred for now.

## Prior/existing features

Existing HTML a/link attributes `rel=alternate hreflang=(language)` can tell the browser there’s a translated version of the current page. Current browsers do not provide any special UI for these. 

* [HTML spec](https://html.spec.whatwg.org/multipage/links.html#rel-alternate): “If the [`alternate`](https://html.spec.whatwg.org/multipage/links.html#rel-alternate) keyword is used with the [`hreflang`](https://html.spec.whatwg.org/multipage/links.html#attr-hyperlink-hreflang) attribute, and that attribute's value differs from the [document element](https://dom.spec.whatwg.org/#document-element)'s [language](https://html.spec.whatwg.org/multipage/dom.html#language), it indicates that the referenced document is a translation.”

Google’s proposal in Web Machine Learning WG:

* [https://github.com/webmachinelearning/translation-api](https://github.com/webmachinelearning/translation-api) Explainer: Translator and Language Detector APIs

## Flaws or limitations in existing features/proposals

The JS-based APIs will force web authors (or extensions) to reimplement what browsers already have, if the purpose is to translate what’s already in the page: the selection translation popup and/or the DOM scanner. Such reimplementation would be more complex or impossible if the DOM tree includes shadow trees, and will not be available for environments without JavaScript.

## Motivation for this explainer, why we think we can do better than the status quo or other proposals

We think APIs designed for in-page translation would be able to solve the use cases with simpler usage while minimizing the exposure of implementation detail and the potential of fingerprinting. We also think focusing on translation to the user's preferred language can achieve simpler APIs. Translation to other languages can be done by the user agent of other users, which are better able to translate content into their preferred language.

## Outline of a proposed solution

1. Add a DOM-based API that will scan the DOM element for translation.  
2. Introduce a way to declaratively select an in-page node for translation.  
3. Introduce a way to open a native translation popup for a given text or a node

## usage, examples, sample code and a prose summary of how it clearly solves the problem(s)

1. Add a DOM-based API: `element.textTranslate()` to optionally show a permission prompt to download the language model and then scan the DOM element for translation.  
   1. This will solve the potential headache of having to manually scan and reconstruct the DOM tree, as the browser already has it.  
   2. We do not need to show prompt for every text translation request, the initial few model downloads might happen seamlessly as long as the total download does not exceed a certain size. After an implementation-specific limit there may be a prompt to proceed further, so that a website cannot stress the storage space and network usage using only the browser agent's resource. This applies to the in-page translation API below too.
2. Introduce a way to declaratively initiate translation of an in-page node via [button command](https://html.spec.whatwg.org/multipage/form-elements.html#attr-button-commandfor) (and thus integrate to Invoker Commands API):   
    `<button type=button commandfor=(id) command=textTranslate>Translate</button>`  
3. Introduce a way to open a native translation popup for a given text or a node:  
    `navigator.openTextTranslate(text or element)`   
   This way either translation webextensions or webpages that want similar experience wouldn't have to reimplement the popup and the language detection/selector, which then the browser doesn't have to expose the corresponding APIs.
