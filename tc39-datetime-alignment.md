# TC39: Datetime Formatting Option Alignment

Author: Eemeli Aro

## User problem to be solved

Active work is happening beyond TC39 in the web space for designing API shapes for datetime formatting.
Users of the web would benefit from the same formatting options
being available in HTML, JavaScript, and Unicode MessageFormat,
as this would make it easier for developers to make use of them.

## Methodology for approaching and evaluating solutions

As mentioned in the context of [site-building ergonomics](https://www.mozilla.org/en-US/about/webvision/full/#site-buildingergonomics),
"The most powerful way to make something easier is to make it simpler,
so we aim to reduce the total complexity that authors need to grapple with to produce their desired result."
Therefore, we should seek to advance a solution that simplifies the developer experience.

Furthermore, as stated under [Extending the Web with JavaScript](https://www.mozilla.org/en-US/about/webvision/full/#extendingthewebwithjavascript),
"we think browsers should learn from what is working, listen to what is needed, and provide the right primitives.
As essential features and abstractions emerge in the ecosystem,
we can standardize and integrate them into the Web Platform directly to make things simpler."
Where possible, the solutions we propose for standardization should not be novel,
but ones that have already proven to work well,
and which integrate well with prior solutions.

## Prior/existing features and proposals

- ECMA-402 already provides two different mutually incompatible datetime formatting option styles:
    1. `dateStyle` and `timeStyle` are
       [Style shortcuts](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat#style_shortcuts)
    2. The [Date-time component options](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat#date-time_component_options)
       provide much more fine-grained control over individual fields.
- [whatwg/html#12591](https://github.com/whatwg/html/issues/12591) is introducing new formatting capabilities
  for the `<time>` HTML element.
- The Unicode MessageFormat standard includes
  [draft functions](https://www.unicode.org/reports/tr35/tr35-78/tr35-messageFormat.html#date-and-time-value-formatting)
  for `:datetime`, `:date`, and `:time` formatting.

## Flaws or limitations in existing features/proposals

The pre-existing ECMA-402 component options support nonsensical combinations, such as:

```js
new Intl.DateTimeFormat('en', { month: 'long', minute: 'numeric' }).format()
// "July at 36"
```

while its style shortcut options are not powerful enough to
specify which parts of the date and time should be included in the result.

This means that as datetime formatting APIs are being developed in other spaces (such as HTML),
it does not necessarily make sense to adopt JavaScript's current API, but to develop something new.

## Motivation for this explainer

If the JavaScript datetime formatting API can be extended to provide a solution
that also works for other parts of the web stack,
the same API can be available for all users.

## Outline of a proposed solution

The Unicode MessageFormat WG spent considerable time in developing its
[draft solution](https://www.unicode.org/reports/tr35/tr35-78/tr35-messageFormat.html#date-and-time-value-formatting),
which is also proposed for the HTML `<time>` element;
this should act as the starting point for the `Intl.DateTimeFormat` option bag design.

Its most significant features are adding the following two options:

- `dateFields`
  - `weekday`
  - `day-weekday`
  - `month-day`
  - `month-day-weekday`
  - `year-month-day` (default)
  - `year-month-day-weekday`
- `timePrecision`
  - `hour`
  - `minute` (default)
  - `second`

These specify the date and time fields that are included in the output,
and could be combined with the pre-existing style shortcuts.

This work should also include the consideration of renaming at least
the `timeZoneName` option as `timeZoneStyle`, to better align with other option names.
Doing so would require making the old name an alias for the new one,
to maintain compatibility with old code.

<!-- ## Shortcomings -->

<!-- ## draft specification -->

## Incubation and standardization destination

TC39
