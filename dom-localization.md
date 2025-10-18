# DOM Localization

Author: [Eemeli Aro](https://github.com/eemeli)  
Previous authors: [Zibi Braniecki](https://github.com/zbraniecki) and [Erik Nordin](https://github.com/nordzilla)

Localization of web content is currently achieved with a multitude of custom solutions,
most of which are unable to express the full width and breadth of human expressions in all languages.

Most localizable content is composed of simple strings,
and most of the content that has some dependency on input variables only uses them as unmodified placeholders.
This has lead to most localization systems having been built up iteratively,
only conservatively adding new features when the need for them has been identified.
Many localization systems used on the web do not support any variance in message patterns,
and the ones that do almost all limit that variance to plurals,
i.e. allowing for patterns like "1 thing" and "3 things" to be separately expressed.

Defining a holistic and complete standard solution for the localization of the web
would allow for developers to not need to pick from a number of less capable localization systems,
and for existing libraries and frameworks to start using the standard solution internally.

## Methodology for approaching & evaluating solutions

This proposal seeks to directly address the concerns raised in the
[Internationalization section of Mozilla's Web Vision](https://www.mozilla.org/en-US/about/webvision/full/#internationalization),
and is in fact directly referenced there:

> For sites which have the resources to invest in localization, we need technology that enables that. Traditionally, translation was accomplished by simply translating blocks of text. Today's rich Web applications can be very dynamic, and thus require substantially more nuance in order to handle genders, multiple nested plural categories, declensions, and other grammatical features which differ across languages and dialects. Moreover, the experience of generating localizations and applying them to sites is error-prone and cumbersome. We want it to be as easy and flexible to localize a site as it is to style it with CSS. Over the past decade we’ve built such a system for Firefox, called Fluent. Beyond localizing Firefox, we’re working with others to bring the ideas and technology behind Fluent to other client-side software projects with the ICU4X Rust library and to the Web with the Unicode Consortium's MessageFormat Working Group. Together with a new proposal for _**DOM localization**_, we see a much more powerful story for localizing Web Sites on the horizon.

Furthermore, when considering what sort of solution is appropriate,
we can help strive towards [The Declarative Web](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb):

> The Web’s basic design is declarative: HTML and CSS have empowered a very wide range of people to develop Web content because they’re easy to understand and allow site authors to focus on the _what_ rather than the _how_. In addition, browser-provided declarative features can guarantee robust & predictable fallback behaviors, in contrast to more fragile imperative approaches, where the responsibility for error handling falls entirely on the developers. Unfortunately, while Web experiences have become substantially more rich over the past 20 years, the expressiveness of HTML and CSS has not kept pace. Instead, authors who want to build interactive applications usually build upon frameworks that then draw their interfaces using HTML but use JavaScript for the heavy lifting of application and rendering logic. For complicated applications, this is a reasonable choice, but the limited HTML/CSS feature set means that authors feel compelled to use larger and increasingly more complex JS libraries & frameworks for nearly any interactive site. In a better world, it would be possible to build more of these experiences using only the capabilities built into the browser.

## Prior proposals that attempt to solve the problem

Previous attempts to add localization support to the web have been focused on its support in JavaScript, as `Intl.MessageFormat`.
This was [first considered](https://web.archive.org/web/20160802003122/http://wiki.ecmascript.org/doku.php?id=globalization:messageformatting) in 2013,
then [briefly discussed](https://github.com/tc39/ecma402/issues/92) again in 2016,
until a [more concerted effort](https://github.com/tc39/ecma402/blob/main/meetings/notes-2019-06-13.md#intlmessageformat) was made to start the actual work,
which became the [MessageFormat Working Group](https://github.com/unicode-org/message-format-wg) under the Unicode CLDR-TC in 2020.

The main reason why earlier efforts here failed to proceed was a general agreement that the available message formats were insufficient.
This has meant that the MessageFormat WG has thus far focused its work on defining
the [Unicode MessageFormat Standard](https://www.unicode.org/reports/tr35/tr35-messageFormat.html),
which is now (since March 2025) a stable part of Unicode's LDML specification.
Thus far, implementations of Unicode MessageFormat are available in Unicode's ICU4C and ICU4J libraries,
as well as the [`messageformat`](https://github.com/messageformat/messageformat/tree/main/mf2/messageformat) JavaScript package.

The JavaScript implementation of Unicode MessageFormat is tracked as
the TC39 [Intl.MessageFormat proposal](https://github.com/tc39/proposal-intl-messageformat).

A [prior version of this DOM localization proposal](https://nordzilla.github.io/dom-l10n-draft-spec/)
was authored by Zibi Braniecki and Erik Nordin,
drawing heavily from experiences in developing, maintaining, and using [Fluent](https://projectfluent.org/),
Mozilla's localization system for natural-sounding translations.
Fluent has also been a significant source of inspiration and experience in the design of Unicode MessageFormat.

The [DOM Overlays rev 3](https://github.com/zbraniecki/fluent-domoverlays-js/wiki/New-Features-(rev-3)) document
identifies and enumerates a number of proposed changes to Mozilla's DOM localization system
that are incorporated in this proposal.

The [DOM Parts proposal](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/DOM-Parts.md)
has some overlap with its aim to enable similar updates to element contents and attributes,
but does not consider localization as one of its use cases.
It also does not provide a mechanism by which locale-specific resources could be loaded.

## Limitations in existing proposals

If we were to not consider DOM localization,
we would leave the standard capability of localizing the web to be a JavaScript feature only.

A number of JavaScript frameworks and libraries for localization exist,
including [one for Unicode MessageFormat](https://github.com/messageformat/messageformat/tree/main/mf2/messageformat).
However, all such solutions -- including the proposed `Intl.MessageFormat` for JavaScript --
require localization to be achieved by extra steps and complications that ought not be necessary.

## Motivation for this explainer

To quote again from [Mozilla's Web Vision](https://www.mozilla.org/en-US/about/webvision/full/#internationalization):

> We want it to be as easy and flexible to localize a site as it is to style it with CSS.

By incorporating localization as a core capability of the web,
we can make it available for everyone.

Through our years of experience in using a Fluent-based variant of DOM localization
in Firefox, Thunderbird, and many other Mozilla products,
we've managed to gain some confidence in the strengths and capabilities of such a solution,
along with finding some pitfalls that a standard solution should avoid.

The composition of HTML, DOM, CSS, and JavaScript and related technologies
is commonly used to create applications with both content and user interface.
One of the core value propositions provided by this stack of technologies is their open, semantic and pluggable nature.

For example, CSS provides technology to apply styles and themes to HTML Documents,
but also enables web browsers and third-party addons such as extensions or accessibility tools
to adjust the styles and themes at runtime.

In a similar way, DOM localization proposes to introduce a localization component to this stack.
This component would allow for HTML documents to be localized in a way that enables web browsers and third-party extensions
to augment documents for the benfit of the user's experience.
Such an open system would allow for construction of localizable web applications that can be introspected for semantic information,
be augmented by external code for different forms of presentation (screen readers, VR etc.),
and be accessible to the global audience.

## Outline of a proposed solution

We propose to introduce a notion of a _**localization context**_,
similar to JavaScript document context, or CSS stylesheets,
which would be composed of a list of _localization resources_ declared in the `<head>` of the document.

A _**localization resource**_ is a file that contains
_**messages**_ that are defined using [Unicode MessageFormat](https://www.unicode.org/reports/tr35/tr35-messageFormat.html).
Similar to stylesheets, developers will be able to programmatically construct any number of _localization resources_,
as well as declaratively define them for HTML documents, shadow DOM trees etc.

Within a _localization context_, each _message_ is identified and referenced with a _**localization identifier**_.
The _messages_ would then be used to localize DOM elements or fragments.

The mechanism to resolve _localization resources_ is nontrivial
and will require a significant amount of up-front design.
The main aspect of the mechanism is that it has to enable the engine to reason about the locales that the user or app requested,
and the locales in which _localization resources_ are available.
It must be able to negotiate between these two sets
to provide an optimal solution for formatting _messages_ in the given _localization context_.

We propose to introduce a set of (potentially namespaced) core attributes to HTML that would allow developers to
declaratively or programmatically bind DOM elements and fragments to localization _messages_.
In addition to a _localization identifier_ reference,
we would also add an attribute for defining _**localization arguments**_
as a set of key-value pairs that serve as arguments when formatting a _message_.

When an element includes a _localization identifier_ reference as an attribute,
its contents and/or other [translatable attribute](https://html.spec.whatwg.org/multipage/dom.html#translatable-attributes) values
are replaced with the corresponding _message's_ formatted results.

In order for the system to be accessible programmatically,
a JavaScript API for the _localization context_ is included in the proposal.
This would be an abstraction built upon the
[Intl.MessageFormat](https://github.com/tc39/proposal-intl-messageformat) object.

We propose that it should be possible for translated _messages_
to refer to or contain DOM elements within the _message_ itself.
For example, a localized _message_ in one locale might like to add `<b>` elements
in appropriate places to make the text bold,
or a _message_ might include a `<button>` or `<a>` link
with some attributes or beahviour defined in HTML or JavaScript.
This is a delicate and nuanced situation that requires heavy scrutiny,
since inserting arbitrary HTML can pose many security risks.

As Unicode MessageFormat only defines the syntax, formatting, and other operations of a single message,
work on DOM localization requires that a web-compatible _message resource_ file format is also defined.
A purpose-built file format enables for Unicode MessageFormat messages to be represented in an easy-to-edit format,
which is not the case when they are e.g. embedded in a JSON document,
and it enables the systematic representation of comments and metadata that is essential for translation work.

Early work on a potential _message resource_ file format provides
a [strawman proposal](https://github.com/eemeli/message-resource-wg) of what such a format could look like,
along with more detailed discussion of why existing file formats are not sufficient.

## Examples and Sample Code

> Note: The following examples include strawman proposals for the additions to HTML, JavaScript,
> and for the _message resource_ file format.
> However, the syntax of a _message_ value is well defined by Unicode MessageFormat.

_Localization resources_ declared in the `<head>` element of a document are aggregated into a _localization context_ that is scoped to that document.

```html
<html>
  <head>
    <link rel="localization" src="uri/for/resource.mf" />
  </head>
  <body>
    <h1 l10n-id="greeting" l10n-args="userName: John"></h1>
  </body>
</html>
```

If we assume that the resolved _localization resource_ contains a _message_ such as:

```ini
greeting = Welcome, {$userName}.
```

Then the above example would produce a DOM matching the following representation after localization:

```html
<html>
  <head>
    <link rel="localization" src="uri/for/resource.mf" />
  </head>
  <body>
    <h1 l10n-id="greeting" l10n-args="userName: John">Welcome, John</h1>
  </body>
</html>
```

It should also be possible to define the _localization resource_ directly within the HTML document:

```html
<html>
  <head>
    <localization> greeting = Welcome, {$userName}. </localization>
  </head>
  <body>
    <h1 l10n-id="greeting" l10n-args="userName: John"></h1>
  </body>
</html>
```

### JS API

Continuing with the same example, changing the header contents would be possible with:

```js
let h1 = document.querySelector("h1");
h1.l10n.set("greeting", { userName: "Mary" });
```

The formatted title would also be accessible directly in JavaScript:

```js
let msg = document.l10n.getMessage("greeting").format({ userName: "Mary" });
```

### Attribute Localization

The localization of [translatable](https://html.spec.whatwg.org/multipage/dom.html#translatable-attributes) element attributes is made possible with:

```html
<html>
  <body>
    <button l10n-id="ok-button"></button>
  </body>
</html>
```

```ini
ok-button = Click me
ok-button.title = Title to show on hover
ok-button.aria-label = Main button
```

Producing the following DOM after localization:

```html
<html>
  <body>
    <button
      l10n-id="ok-button"
      title="Title to show on hover"
      aria-label="Main button"
    >
      Click me
    </button>
  </body>
</html>
```

Such atomic binding between a UI widget and a composed localization unit
would be particularly useful for localization of Web Components
where rich set of attributes could be used to carry localization _messages_
across the boundary from the document to shadow DOM.

This binding would also enable locale consistency for the whole UI element
ensuring that content and attributes of each element are localized into the same locale,
be it the primary locale, or any fallback locale.

### DOM Overlays

#### Text-level Elements

_Messages_ should be able to include [text-level elements](https://html.spec.whatwg.org/multipage/text-level-semantics.html)
that would be included in the localized results.
For example, a developer may create a paragraph element such as the following.

```html
<p l10n-id="key1">Message to be localized.</p>
```

A translator should be able to provide a translation that contains text-level elements if desired.

```ini
key1 = This is {#em}my{/em} localized message.
```

At which point, the final localized DOM would be equivalent to the following,
in which the `<em>` element is interpreted correctly as HTML and rendered accordingly.

```html
<p l10n-id="key1">This is <em>my</em> localized message.</p>
```

#### Functional Elements

Functional elements such as `<img>` and `<a>` may have attributes for which the developer
desires a translator to provide localized translations.

For example, a developer may provide an `<img>` with a `src` attribute.

```html
<p l10n-id="key2">Hi, <img src="world.png" /></p>
```

A translator should be able to provide an `alt` attribute for the `<img>` element.

```ini
key2 = Hello, {#img alt=|world|}!
```

The above example would produce the following DOM after localization:

```html
<p l10n-id="key2">Hello, <img src="world.png" alt="world" />!</p>
```

This example allows the translator to add the `alt` attribute unconstrained,
but it will likely be important to design syntax such that the translator can only add or modify
a localized attribute if it is explicitly requested by the developer.
This would ensure that the translator cannot override an attribute
that is not meant to be localized, such as `src` or `href`,
unless the translator has explicit permission to do so.

#### Structural Elements

A developer may want to have `<ul>` or `<ol>` elements be part of the localization,
in which the translator can then add `<li>` elements.

```html
<div l10n-id="key3">
  <ul></ul>
</div>
```

A translator should then be able to provide a localized message along with list items.

```properties
key3 = This is a localized list:
  {#ul}
    {#li}Localized item 1{/li}
    {#li}Localized item 2{/li}
    {#li}Localized item 3{/li}
  {/ul}
```

The above example would produce the following DOM after localization:

```html
<div l10n-id="key3">
  This is a localized list:
  <ul>
    <li>Localized item 1</li>
    <li>Localized item 2</li>
    <li>Localized item 3</li>
  </ul>
</div>
```

## Caveats and Shortcomings, and other drawbacks of design choices, both current and any prior iterations

### Resource Loading May be Slow

_Localization resources_ that are loaded from external files will need to be processed asynchronously.
This means that document rendering could experience a "flash of unlocalized content",
much like a [flash of unstyled content](https://en.wikipedia.org/wiki/Flash_of_unstyled_content).
This will need to be similarly addressed.

### Fallback Behaviour Needs Consideration

As localization can only be done after a _message_ is first defined for the source language,
it's quite common for a document to need to be rendered when no appropriate translation of a _message_ is available.
It should therefore be possible for the fallback order of localizations to be well defined.
This fallbacking can at times be a multi-stage process, such as `fr-CH, fr, en`,
where a Swiss French localization may rely on _messages_ from a generic French localization
before falling back to an English localization.

The DOM localization system used for Firefox allows for the need to perform fallback to be determined relatively late,
when attempting for format a _message_ that may not be available in the preferred locale.
This capability requires being able to load new _localization resources_ before completing the formatting,
which makes each formatting call asynchronous.

It may be preferable to leave locale fallback out of the capabilities of DOM localization,
and to only ensure that interfaces are provided that make it possible to implement with JavaScript.

### Localizable Attributes May Need to be Defined Explicitly

Regarding the localization of attributes either directly or via DOM overlays,
we may want to adopt an opt-in-only model in which a developer
must explicitly list which attributes should be localizable,
rather than making all [translatable attributes](https://html.spec.whatwg.org/multipage/dom.html#translatable-attributes) localizable by default.

For example, in the following example,
`alt` is the only attribute that localizers would be able to modify in translation:

```html
<p l10n-id="key2" l10n-attrs="alt">Hi, <img src="world.png" /></p>
```

### Element Identify Should be Retained

Elements should not be recreated during localization.

For example, even after multiple retranslations,
the identity of the following `<button>` element should be the same:

```html
<div data-l10n-id="submit-form">
  <button id="submit">Submit</button>
</div>
```

```properties
submit_form = {#button}Submit{/button}
```

<!-- ## Draft Specification -->

## Incubation and/or Standardization Destination

Incubation could be in the W3C i18n WG, standardization in WHATWG HTML.
