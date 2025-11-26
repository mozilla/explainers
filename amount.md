# Amount explainer

Author: [Tantek Çelik](https://tantek.com/)

## User problems to be solved

On the web, users from around the world often see numerical amounts on web pages either in unfamiliar units, 
or with unfamiliar punctuation denoting large or decimal amounts which can either be hard to interpret, or worse, easily misinterpreted. 
If publishers had a mechanism to explicitly denote these numerical amounts and their units if any, 
then browsers could offer a privacy-respecting user-interface to display these amounts to the user according to their local units and punctuation preferences.

## Methodology for approaching and evaluating solutions

Per the priority of constituencies, we should first look for solutions that users can trust, in both fidelity and privacy. 

1. Any unit conversions need to be consistent and reliable, like a calculator, as users will expect simple arithmetic, 
   perhaps allowing only negligible rounding errors. 
   For example, that eliminates currency conversion, which is variable and not tolerant of rounding errors.
2. Conversions that might occur due to localization preferences, such as conversions from feet to meters,
   need to not be observable to sites that are not already aware of the preference.
   For example, a conversion to preferred units might be implemented through the use of tooltips or another purely-browser controlled interface.

Second, we should look for solutions that are minimal work, simpler, and more dependable and robust for developers. 
In particular, we should look for and prefer [declarative approaches](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb) over imperative approaches. 

Out of scope: 
* currencies (for this Explainer), because their conversions are subjective and changing (not 100% reliable). We may still consider the backward compatible extensibility of any solutions, in order to enable a future Explainer to add currency support. We may also consider how solutions could treat a currency amount as a numerical amount with an unknown unit, for the purposes of converting at least the numerical portion to user preferred or locale-specific punctuation for large or decimal numbers.

## Existing features and prior proposals
There is a broad range from implemented features to various prior and in-progress proposals that address or attempt to address some to all of the user problems noted above. 
This section documents them, roughly clustered into well implemented existing standards features, 
to proposals that may have some to no partial or experimental support from publishers or perhaps a single implementation.

### Existing features
In HTML we have the `<time>` element which is useful for publishing date time information, 
and in particular [durations](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#valid-duration-string) 
amounts with units of seconds, minutes, hours, or days.

HTML also has the [`<data>` element](https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-data-element) 
which can be used to represent numerical amounts independent of punctuation, e.g.:

`<data value=1000 lang=en>1,000</data>`
`<data value=1.5 lang=fr>1,5</data>`

Both elements are used in the wild on web pages. 
There are techniques for using the [time element to provide accessible “ago“ durations](https://shkspr.mobi/blog/2020/12/making-time-more-accessible/), 
however there are no known examples in the wild of using data elements for numerical amounts with punctuation.

Both elements are also widely natively supported by browsers 
([MDN data support](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/data#browser_compatibility), 
[MDN time support](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/time#browser_compatibility)), 
however without any default user-discernible effects, 
including [in screen readers]([https://twitter.com/LeonieWatson/status/1333078194925264898](https://web.archive.org/web/20201129160223/https://twitter.com/LeonieWatson/status/1333078194925264898)), 
despite the “[First rule of ARIA use](https://www.w3.org/TR/aria-in-html/#rule1)” encouraging publishers to use native HTML semantic elements.

### Prior proposals
The microformats community did a lot of [research into measure formats](https://microformats.org/wiki/measure) 
for amounts with physical units in the mid-2000s, 
culminating in an [hmeasure draft proposal](https://microformats.org/wiki/measure-brainstorming#Draft_Schema), 
which was eventually evolved in the 2010s into an [h-measure proposal](https://microformats.org/wiki/measure-brainstorming#microformats2) 
in modern microformats2 syntax (which itself is widely used in various blogging services, software, and plugins).

In [2019](https://webwewant.fyi/events/2019-wordcamp-us/) the WebWeWant community submitted a “want” (recognized as a “Judges’ Pick”) 
for “[browsers to localize data like dates and numbers](https://webwewant.fyi/wants/59/)” 
([GitHub discussion](https://github.com/WebWeWant/webwewant.fyi/discussions/188)). 

The “want” proposed a default browser user interface enhancement of the standard HTML `<time>` element 
and a new HTML `<amount>` element with a `units` attribute for currency or physical units. 
In addition there were unclear proposal-by-example attributes for numeric `decimals`
(perhaps indicating significant digits or number of optional digits after the decimal point), 
and a boolean `non-zero-decimals` which may be an attempt to only display non-zero decimals
(thus a `false` value meaning always display decimals, which seems to be an author-unfriendly double-negative expression). 
Also the second `<amount>` example is missing a `value` element which we can infer from context should have been `value="2"`.

The idea for an HTML `<amount>` element with a `units` attribute overall seems promising and worth exploring, expanding, and specifying.

In 2022, a web developer [proposed](https://github.com/openui/open-ui/issues/499) an 
[m or measure element](https://github.com/futekov/measure-element), with `unit` and `value` attributes, 
and included research and analysis of the aformentioned microformats measure efforts, 
as well as existing HTML elements noted above, and the Wikipedia `{{convert}}` template as prior art. 
The proposal also includes a way for the author to express their confidence in the "convertiblilty" of 
any particular measure with a `convert` attribute that takes values `never|auto|eager`. 
As a result of this proposal, the Open UI Community Group 
[resolved to incubate a solution to semantic measurements](https://github.com/openui/open-ui/issues/499#issuecomment-1084962904) without committing to any particular existing solution or proposal, 
thus indicating a likely receptive candidate incubation destination for this explainer.

There is a late 2024 [measure proposal](https://github.com/tc39-transfer/proposal-measure) in TC39 that, 
despite the imperative approach, may have some additional useful (researched) semantics to consider 
in a proposed declarative amount solution, such as compound units and separate major and minor units.

## Flaws or limitations in existing features and prior proposals
* The HTML `<time>` element is limited to representing temporal amounts, and would be inappropriate for other physical or numerical amounts.
* The HTML `<data>` element is limited to representing unitless numerical amounts.
* The `h-measure` proposed microformat while cleverly re-uses `<data>` elements, requires multiple (3-4) elements to represent a single numerical amount.
  Multiple elements for a single amount are a substantially lengthier syntax,
  both suboptimal for developer ergonomics and easier to get wrong or break in maintenance.
* The WebWeWant `<amount>` proposal is both incomplete, and yet seems to also have unnecessary features like the boolean `non-zero-decimals`.
* The `<m>` or measure element proposal has many good attributes (in both senses),
  and may present an opportunity for collaboration. The only obvious nit is in the name,
  as a “measure” implies something was measured, which we cannot (do not want to) assume.
  We want to enable amounts of any type and don’t want to imply that a measurement was done in all potential use cases.
  An amount could be a measure or an abstract quantity or part of a request or rule,
  such as legislated speed limits, which exist regardless and independent of any particular measurement.
* The TC39 measure proposal defines imperative (scripting) features,
  while we prefer to define a declarative mechanism (as noted in the Methodology section above),
  that can be captured in markup.
  In particular, the expression of semantics (meaning) in the web platform
  should be encoded in markup, without depending on script to function.
  A script-only proposal would also create worse developer ergonomics (wordier and more fragile imperative syntax)
  than the existing `h-measure` microformat.
  Additionally, like the measure element proposal, the term "measure" implies a narrower meaning than potential use-cases.

## Motivation for this explainer
By choosing the best from prior proposals, we can create a simpler, more ergonomic and robust solution for web developers that starts with solving the most common use-cases, with room for extensions to solve more use-cases as necessary.

By taking inspiration from the prior declarative features and proposals in particular, we can create a solution for [the declarative web](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb) for all the reasons that such solutions are better than procedural or imperative approaches.

## Outline of proposed solution
Proposal: a new HTML `<amount>` element (similar to `<data>` and `<time>`), with:
* `value` attribute: optional numerical value, similar to the `<data>` element’s `value` attribute. Otherwise element contents are parsed for a numerical value.
  * E.g. `<amount>42.2</amount>` or `<amount value=42.2 lang=de>42,2</amount>`
* `unit` attribute: optional physical ([SI](https://en.wikipedia.org/wiki/International_System_of_Units)) unit to express measures.
  Absent a `unit` attribute, the element contents are parsed for a numerical value, and the remaining portion of the element contents is parsed for an SI unit.
  Simple compound SI units (e.g. `km/h`) are common enough in web content to also merit supporting.
  User agents may fully algorithmically transform such a unit to the user’s locale (user opt-in) or other display preferences.
  * E.g. `<amount>42.2km</amount>` or `<amount value=42.2 unit=km> lang=de>42,2km</amount>`
* If there is no `unit` attribute, and no valid unit parsed from the element contents, the amount is unitless and expresses a quantity.
  User agents may format (user opt-in) large and/or decimal quantities in locale-specific
  [tridigit](https://en.wikipedia.org/wiki/Decimal_separator#Digit_grouping) and
  [decimal separators](https://en.wikipedia.org/wiki/Decimal_separator) (e.g. ',' vs '.' etc.),
  taking care to preserve any script observable metrics to avoid exposing locale information.

### currency extension
As an exploration, to extend the `<amount>` element to support currency, we could do so with:
* `currency` attribute: optional currency code ([ISO 4217](https://en.wikipedia.org/wiki/ISO_4217)) to express a monetary amount. User agents could use an online exchange service to compute rough equivalents and note the source and time dependency (temporal context) of any such conversions.
  * e.g. `<amount currency=USD value=64000>$64,000</amount>`
* The presence of both `unit` and `currency` attributes is an error. The user agent must ignore both attributes.
* Quantity change: If neither a unit (attribute or parsed) nor `currency` attribute are present, the amount is unitless and expresses a quantity.

## Alternatives considered

* Add a unit="" attribute to the `<data>` element (proposed in [#42](https://github.com/mozilla/explainers/issues/42))
  * Advantage: seemingly simpler: only adding an attribute to an existing element rather than adding a new element
  * Disadvantage: likely to violate existing developer presentational and behavioral expectations of the existing `<data>` element, that is no (browser default) change in presentation or behavior is ever expected on any `<data>` element. A key goal this functionality (through the `<amount>` element) is to explicitly encourage both a default presentational hint (e.g. from browser default styling) and a behavior change (e.g. one or more context menu items or other UI affordance) to semi-automatically provide unit conversions to the user. While this is theoretically somewhat implementable via an attribute selector in the default style sheet, the change from `<data>` elements never by default impact presentation and behavior to `<data>` elements may sometimes impact presentation and behavior adds complexity to the developer mental model of the existing `<data>` element.
  * Conclusion: per the principle of least surprise, it is better to leave the `<data>` element alone, and predictable in its prior and well-established default (lack of any special) behavior, and instead provide web authors with an element that explicitly enables browser default enhancements.
