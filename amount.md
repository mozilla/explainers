# Amount explainer

Author: [Tantek Çelik](https://tantek.com/)

## User problems to be solved

On the web, users from around the world often see numerical amounts on web pages either in unfamiliar units, 
or with unfamiliar punctuation denoting large or decimal amounts which can either be hard to interpret, or worse, easily misinterpreted. 
If publishers had a mechanism to explicitly denote these numerical amounts and their units if any, 
then browsers could offer a privacy-respecting user-interface to display these amounts to the user according to their local units and punctuation preferences.

## Methodology for approaching & evaluating solutions

Per the priority of constituencies, we should first look for solutions that users can trust, 
e.g. if a solution enables an affordance to convert units, it should be 100% reliable like a calculator, 
as users will expect any unit conversions to be simple arithmetic.
Second, we should look for solutions that are minimal work, simpler, and more dependable and robust for developers. 
In particular, we should look for and prefer [declarative approaches](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb) over imperative approaches. 

Out of scope: 
* currencies (for this Explainer), because their conversions are subjective and changing (not 100% reliable). We may still consider the backward compatible extensibility of any solutions, in order to enable a future Explainer to add currency support. We may also consider how solutions could treat a currency amount as a numerical amount with an unknown unit, for the purposes of converting at least the numerical portion to user preferred or locale-specific punctuation for large or decimal numbers.

## Prior features and existing proposals

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
including [in screen readers](https://twitter.com/LeonieWatson/status/1333078194925264898), 
despite the “[First rule of ARIA use](https://www.w3.org/TR/aria-in-html/#rule1)” encouraging publishers to use native HTML semantic elements.

The microformats community did a lot of [research into measure formats](https://microformats.org/wiki/measure) 
for amounts with physical units in the mid-2000s, 
culminating in an [hmeasure draft proposal](https://microformats.org/wiki/measure-brainstorming#Draft_Schema), 
which was eventually evolved in the 2010s into an [h-measure proposal](https://microformats.org/wiki/measure-brainstorming#microformats2) 
in modern microformats2 syntax (which itself is widely used in various blogging services, software, and plugins).

In 2020 the WebWeWant community submitted a “want” (recognized as a “Judges’ Pick”) 
for “[browsers to localize data like dates and numbers](https://webwewant.fyi/wants/59/)” 
([GitHub discussion](https://github.com/WebWeWant/webwewant.fyi/discussions/188)). 

The “want” proposed a default browser user interface enhancement of the standard HTML `<time>` element 
and a new HTML `<amount>` element with a `units` attribute for currency or physical units. 
In addition there were unclear proposal-by-example attributes for numeric `decimals`
(perhaps indicating significant digits or number of optional digits after the decimal point), 
and a boolean `non-zero-decimals` which may be an attempt to only display non-zero decimals
(thus a `false` value meaning always display decimals, which seems to be an author-unfriendly double-negative expression). 
Also the second `<amount>` example is missing a `value` element which we can infer from context should have been `value="2"`.

...
