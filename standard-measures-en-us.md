# Standard Measures with U.S. English

Author: Henri Sivonen

## User Problems to be Solved

Users who don't have a strong connection to U.S. measurement, date formatting, and other similar conventions use software with U.S. English as the UI language. This is a problem when the users, for example, don't want to see temperatures in Fahrenheit or accidentally book tickets or accommodation for the wrong days due to site-supplied date pickers (as opposed to browser-provided `&lt;input type=date>`) putting Sunday in the first column when presenting a month as a grid (especially when neither the user nor the site is U.S.-based: e.g. when a user from one EU country is booking train tickets in another EU country).

We believe this problem can be addressed by the browser prepending a language tag with a special region code (provisionally `en-ZZ`, "English of Unknown Region") to the language preference list when `en-US` would otherwise be the first item on the list and the browser determines that standard measures would work better for the user.


### Problem Details

For much of the software industry, including major browsers and operating systems, U.S. English is the source UI language that other languages are translated from. Also, for multiple reasons (some related to U.S. English being the translation source and some not), it is common for users whose primary language is not U.S. English to run software, including Web browsers, with U.S. English as the UI language.

This is common enough to stand out as a distinct phenomenon that relates to U.S. English specifically, which is why this explainer is about that specific case. Only 29% of unique Firefox telemetry submitters using `en-US` as the UI language are in the United States and when looking at Firefox UI language in the world excluding the United States, 33% of unique telemetry submitters use the `en-US` configuration.

Non-linguistic aspects of localization, such as measurement units, datetime formats, other calendaring aspects like what weekday goes in the first column when a month is presented as a grid, default paper size, etc. tend to depend on *region*. However, the concept of *locale* for such non-linguistic purposes and the concept of *user interface language* are coupled: `en-US` as the user interface language results in `US` being taken as the region for the non-linguistic aspects of localization.

(For the region-dependent parameters that are primarily at issue here, see [supplementalData.xml](https://github.com/unicode-org/cldr/blob/main/common/supplemental/supplementalData.xml) and [units.xml](https://github.com/unicode-org/cldr/blob/main/common/supplemental/units.xml) in CLDR.)

While the linguistic aspects of running software in U.S. English even when it is not the user’s primary language or when the user is not in the U.S. context otherwise are evidently OK enough for users who do so, the non-linguistic implications of United States as a region result in considerably less suitable, often annoying and confusing, outcomes for users who don’t actually have a strong connection to U.S. measurement conventions. This is due to measurement conventions of the United States differing maximally from international standards or what is common practice elsewhere.

For illustration of how exceptional the United States is: [CLDR carries data about what units are used for various measurements around the world](https://github.com/unicode-org/cldr/blob/main/common/supplemental/units.xml). In the cases where there isn’t a single world-wide unit for a given measurement type (e.g. barrel for oil applies world-wide), in all cases but one (blood glucose) the United States is an exceptional region. In the case of blood glucose, the list of exceptional regions is exceptionally long and it’s fair to ask how much longer the exception list would need to be for the default and the exception to flip around.

Thus, when the user is running software in U.S. English (for whatever reason) but has no strong connection to the maximally unusual U.S. measurement (etc.) conventions, the non-linguistic localization outcomes flowing from the browser reporting `en-US` as the user’s language/locale go wrong from the user’s perspective.


## Outline of a Proposed Solution

In CLDR proper or, failing that, in browser-tailored CLDR, define the semantics of `en-ZZ` (English of Unknown Region) as U.S. English with standard measures. (The primary aspect of the proposed solution is that the region code fits in the two-ASCII-letter space. Using `ZZ` specifically is not essential.)

With CLDR data, the “likely subtags” operation already expands `en` to `en-Latn-US`, so it’s defensible to argue for the U.S. language variant here (but see the “Open Questions” section about British English being the catch-all English cluster for language matching in CLDR). (Changing the likely subtags operation itself so that `en` would expand to `en-Latn-ZZ` would likely be too disruptive for various existing users of CLDR, so a change to the operation itself is _not_ proposed here.)

`en-001` in CLDR already uses more British-like spelling, which makes `en-001` unsuitable as-is. Also, `ZZ` stays within the format of two ASCII letters, which is expected to be more compatible with systems whose authors haven’t thought of three-digit region subtags. `ZZ` is already defined as Unknown Region, so this does not involve minting a new code in the two-letter value space.

In the case of most CLDR measurement convention data, `en-ZZ` could fall back to `001` without introducing new data. However, there are some specific situations where `001` does not reflect international standards but it would be appropriate for `en-ZZ` to do so.

Specifically, [the week numbering rule for `001` does not use ISO weeks](https://unicode-org.atlassian.net/browse/CLDR-18506), but it tends to be the case that the locales where week numbers are actually prominently used use ISO weeks. This is why the HTML week input field supports only ISO weeks and why `en-ZZ` should imply ISO weeks.

Also, to avoid ambiguity, the short date format should be `yyyy-MM-dd` for the Gregorian calendar.

When the first item on the user’s language preference list (as currently determined) is `en-US`, have the browser prepend `en-ZZ` to the list (i.e. `en-ZZ` becomes navigator.language and the ECMA-402 fallback locale) if the browser determines, using heuristics to be defined (or by explicit override by the user), that the user appears to prefer international standard measures.


## Methodology

The solution needs to be suitable for browsers to activate automatically if the browser infers from some (likely operating system-level) signals that the user is likely not operating in the U.S. measurement context despite having the UI language set to U.S. English.

If the user had to take a specific action in browser preferences in order to opt into international standard measures, the utility of the feature would be greatly diminished due to lack of discoverability: It’s already too easy for users to get resigned to U.S. English implying measures etc. not being internationally standard ones and the user now knowing what to do about it in the context of whatever reason causes them to use U.S. English as the UI language despite being annoyed or confused by the measurement implications.

Given that the solution needs to be suitable for automatic activation by browsers, its privacy implications need to be minimal enough to make automatic activation appropriate from the privacy perspective.

This is about trying to have both “[Everyone’s activity on the Web should be private by default.](https://www.mozilla.org/en-US/about/webvision/full/#privacy)” and the feature, which can be seen as the flip side of Mozilla’s Web Vision asking [sites to expose i18n-relevant semantics](https://www.mozilla.org/en-US/about/webvision/full/#internationalization) to the browser (this is about the browser exposing user preferences, but in a very narrow way, to sites), actually working for a substantial portion of the user base as opposed to only those who know to flip a browser-level opt-in (in addition to configuring the operating system layer in a way that signals a preference for standard measures to native apps).

Given that the solution needs to be suitable for automatic activation by browsers, it must avoid suggesting to Web sites that they use a non-U.S. regional variant of English in order to avoid getting rejected by users who specifically prefer U.S. English and have the UI language set that way for that reason.

Given that the solution needs to be suitable for automatic activation by browsers, its Web compatibility risk needs to be minimal. (“[While not absolute, backwards-compatibility is a key principle of the Web: as new capabilities are added (typically as extensions to the core architecture), existing sites are still usable.](https://www.mozilla.org/en-US/about/webvision/full/#openness)”)


## Prior Proposals and Existing Specs

* We should endeavor to remove the reasons for non-U.S. users to use U.S. English as the UI language.
* Users should use “International English” / “World English” / `en-001` instead of U.S. English.
* Users should use a pre-existing regional variant of English for a non-U.S. region that’s (as far as CLDR data goes) better aligned with international standards, notably Ireland.
* Pick an existing actual standard-using non-English region as a special region (seen with [`en_DK` in glibc](https://sourceware.org/git/?p=glibc.git;a=blob_plain;f=localedata/locales/en_DK;hb=refs/heads/master))
* Use the [Region Override](https://www.unicode.org/reports/tr35/tr35.html#RegionOverride) Unicode BCP 47 U Extension to communicate the user’s actual region. (E.g. `en-US-u-rg-fizzzz` for U.S. English language with the non-linguistic conventions for Finland.)
* Use feature-specific [Unicode BCP 47 U Extensions](https://www.unicode.org/reports/tr35/tr35.html#unicode-bcp-47-u-extension). (E.g. `en-US-u-fw-mon-u-hc-h23-u-ms-metric-u-mu-celsius` for U.S. English language with weeks starting on Monday, 24-hour clock, metric measurements, and celsius temperatures.)
* Use feature-specific Unicode BCP 47 U Extension fields and values but out-of-band relative to Accept-Language and navigator.language. (I.e. the pre-existing API/protocol fields carrying a BCP 47 language tag would remain as language-region tags and the U Extension data would be exposed separately.)
    * This is the general approach of the previous [User Locale Preferences](https://github.com/romulocintra/user-locale-client-hints) proposal and the [Locale Extensions](https://github.com/ben-allen/locale-extensions/blob/main/README.md) ([s-p: negative](https://github.com/mozilla/standards-positions/pull/985)) proposal that superceded it.

## Flaws of Prior Proposals and Existing Specifications

### Removing reasons for non-U.S. users to use U.S. English as the UI language

While endeavoring to remove reasons for non-U.S. users to use U.S. English as the UI language is aligned with [Mozilla’s Web Vision](https://www.mozilla.org/en-US/about/webvision/full/#internationalization), the reasons why users run software with U.S. English as the UI language even if they aren’t in the U.S. context are numerous and varied enough that telling users not to or trying to remove all the reasons are not viable solutions: Even when making progress on some points, there’s is always something still more why some users run software with U.S. English as the UI language. Also, this approach has, in principle, been available to attempt since before the Web existed, but the reasons for non-U.S. users to run software with U.S. English as the UI language haven’t been removed by now and new ones show up.

For an illustration of the variety of reasons why non-U.S. users run software with U.S. English as the UI language, here is a non-exhaustive list of example reasons in no particular order:

* Whole-system consistency when it’s not feasible to get the native language across all apps.
* Having error messages and UI labels that yield better search results on the Web.
* Matching UI strings to teaching materials.
* Being able to write about the software being used to an international audience.
* Avoiding mistranslations that are functionally confusing (e.g. "on" and "off" getting their labels swapped!).
* Avoiding the indignity of having to look at a low-quality translation (e.g. a button to create a new item labeled the way a release note section about new features would be titled!).
* Running development versions of software that don’t have the translations immediately in sync.
* Running release versions of software whose localizers have quit before the current version so the current release shows mixed-language UI.
* The IT department of an organization wishing to have a single system image for the whole organization, which includes people who can’t (at all or properly) read the main language of the region the organization is in.
* The system not supporting the user’s native language for text-to-speech and the user wishing the system to read UI text out loud.
* The system not supporting the user’s native language for speech recognition and the user wishing to be able to refer to UI items by speech.
* The system coupling the language of writing tools to UI language.
* In systems that allow arbitrary region for English for the regions were added for English in [https://unicode-org.atlassian.net/browse/CLDR-8642](https://unicode-org.atlassian.net/browse/CLDR-8642) and its follow-ups, choosing the actual region likely makes the user stand out as a privacy matter, and the user can assume a larger anonymity set by choosing U.S. English.


### Using a Non-U.S. Variant of English

In some cases, this option has been available for a long time, but it still hasn’t removed the problem to be solved. That this hasn’t solved the problem by now should alone suffice as an observation why this approach is flawed as a solution in practice. However, here are additional points why this approach is flawed:

* When the user’s use of U.S. English is about using the untranslated UI strings, this does not strictly accomplish it.
* This approach isn’t consistently available. For example, Firefox is offered for download in “English (US)”, “English (British)”, and “English (Canadian)” but to get both international standard units and ISO weeks, you’d need “English (Ireland)”, which is not available.
* To the extent users use U.S. English over another variant of English because they specifically prefer U.S. spelling, vocabulary, or pronunciation in the UI, the regional variants that result in international standard measures also affect these linguistic aspects.
* To the extent foreign learners of English are taught about regional variants of English, they tend to be taught about the general characteristics of U.S. English in contrast to the general characteristics of British English but not about other regional variants in enough detail to understand what choosing another variant would really entail. It is likely that non-English-native users don't know what they would be getting themselves into if they chose “International English” or a regional variant other than U.S. English or British English.
    * British English does not solve the measurement issue, as the UK also uses measures that differ from what is common globally. (There are three possible values for a Unicode Measurement System Identifier: `metric`, `ussystem`, and `uksystem`.)
* In the case of “International English” or “World English”, there is the additional Web compatibility concern that Web sites may expect the region subtag, if present, to consist of two ASCII letters and might be intolerant of the region subtag of `en-001` consisting of three digits. (See e.g. [https://unicode-org.atlassian.net/issues/CLDR-9429](https://unicode-org.atlassian.net/issues/CLDR-9429) where someone’s code didn’t anticipate `ar-001` in a non-Web context.)
* `en-001` does not have ISO weeks in CLDR. (Though arguably it should.)
* Anecdotally, it seems that users shy away from `en_DK` when installing desktop Linux even if they know what it is for.


### Use the [Region Override](https://www.unicode.org/reports/tr35/tr35.html#RegionOverride) Unicode BCP 47 U Extension

This has notable problems:

* Exposing the actual region would expose an unnecessary amount of fingerprinting bits, since the actual region subdivides the anonymity set into many more subsets than it needs to be subdivided into for the purpose of communicating that the user prefers international standard measures.
* The first problem could be addressed by using a single region identifier instead of the actual region identifier to signal “international measures”, but choosing such a region identifier but users might not be politically comfortable with browsers automatically conjuring up a region identifier associated with an actual region (other than “world” or “unknown”) for them if they have no connection to that region and `001` does not have ISO weeks in CLDR.
* When the short date format associated with the actual region swaps day and month around relative to the U.S. short date format, there would be no indication whether a given date rendered to the user is swapped or not. (In contrast, if the non-U.S. date format is `yyyy-MM-dd`, this problem does not arise, since the distinction is clear.)
* Extension tags are a Web compatibility risk if communicated in API or protocol fields where Web sites have a historical reason to expect to see only a lone language subtag or a language subtag, hyphen, and a region subtag.


### Use feature-specific [Unicode BCP 47 U Extensions](https://www.unicode.org/reports/tr35/tr35.html#unicode-bcp-47-u-extension)

This solution has at least these problems:

* It still exposes excessive fingerprinting bits unless only one combination of these extensions is exposed.
* It covers only a small number of measurement-like things. Most obviously, it does not cover the short date format, which would be useful to override to `yyyy-MM-dd` for clarity.
* Like the previous item, it has the Web compatibility risk of introducing extension tags where Web sites might not be prepared to deal with them.


### Use feature-specific Unicode BCP 47 U Extension fields and values but out-of-band relative to pre-existing API/protocol fields

This addresses the Web compatibility problem relative to the previous item, but does not address the other problems with it. (As noted, this is the general approach of the [User Locale Preferences](https://github.com/romulocintra/user-locale-client-hints) proposal and the [Locale Extensions](https://github.com/ben-allen/locale-extensions/blob/main/README.md) ([s-p: negative](https://github.com/mozilla/standards-positions/pull/985)) proposal that superseded it.)


## Motivation for This Explainer

In general, the User Problem to be Solved described at the top of this document is annoying enough for the affected users that it seems worthwhile to try to solve it. (On the flip side, the problem probably isn’t bad enough for users to take specific action to solve it; see below about the solution needing to be automatic.)

As for why now, this Explainer is motivated as an alternative approach to proposals that have already been brought forward recently ([User Locale Preferences](https://github.com/romulocintra/user-locale-client-hints) and [Locale Extensions](https://github.com/ben-allen/locale-extensions/blob/main/README.md)) and in the context of ECMA TC39 TG1 feedback to TG2 that various API already defined in or proposed for ECMA-402 are substantially less useful than they are intended to be unless this problem is solved.


## Open Questions

The proposed solution above refers to “heuristics to be defined”, so actually implementable heuristics need to be developed. These would likely involve seeing if the operating system reports values that don’t match the implications of the US region for measurement-related system settings.

It is unclear this feature would be accepted in regions where weeks start on Sunday but that use celsius and metric units. If this feature was enabled, site-rendered grid-based date pickers putting Monday in the first column could become confusing for users whose local convention happens to match the U.S. convention of putting Sunday in the first column. Arguably the layout of date pickers not matching user expectations is more confusing than e.g. seeing temperatures in Fahrenheit: The former can cause unnoticed misbooking but when seeing a Fahrenheit temperature, the failure mode is fairly obvious to the user.

Experimentation is needed to see if `en-ZZ` is sufficiently Web-compatible in the sense of not having unwanted linguistic side effects on sites that don’t have knowledge of this proposal. A Web site that processes a language preference list (from `Accept-Language` or `navigator.languages`) correctly is supposed skip over `en-ZZ` and look at the next item on the list, `en-US`.

How Web sites that look at only `navigator.language` will behave is more of a risk.

When a region is absent, the CLDR “[likely subtags](https://github.com/unicode-org/cldr/blob/main/common/supplemental/likelySubtags.xml)” operation expands `en` to `en-Latn-US` (with the exception of `en-Shaw`, which expands to `en-Shaw-GB`), so a system that throws away an unsupported region and then applies “likely subtags” would end up with `US` as the effective region.

However, LDML defines [Enhanced Language Matching](https://www.unicode.org/reports/tr35/tr35-65/tr35.html#EnhancedLanguageMatching) with the concept of `paradigmLocales` that divides English into two paradigm clusters: `en` (i.e. U.S. English consistent with likely subtags above) and `en-GB`, and CLDR has the rules `&lt;matchVariable id="$enUS" value="AS+CA+GU+MH+MP+PH+PR+UM+US+VI"/>` and `&lt;languageMatch desired="en_*_$!enUS"	supported="en_*_GB"	distance="3"/>	&lt;!--  Make en_GB preferred... -->`. That is, the U.S. English cluster enumerates regions and all other regions are considered to belong to the British English cluster. This means that an application that has U.S. English and British English available, that does not have data for `en-ZZ`, and that implements this part of LDML with currently-existing CLDR data will choose `en-GB` when `en-ZZ` is requested. (The British English cluster was made preferred over the U.S. English cluster in [https://github.com/unicode-org/cldr/commit/d131727eba3a813023ce6622558cf54d9e819935](https://github.com/unicode-org/cldr/commit/d131727eba3a813023ce6622558cf54d9e819935) , which fixed [https://unicode-org.atlassian.net/browse/CLDR-10148](https://unicode-org.atlassian.net/browse/CLDR-10148) , which requested making a request for `en-GB` pick `en-IN` from the available options of `en-US` and `en-IN`.)

And, obviously, sites that implement language matching without reference to LDML/CLDR can treat a request for `en-ZZ` in whichever way.

One option would be to make an exception to the relationship of `navigator.language` and `navigator.languages`/`Accept-Language`/ECMA-402 fallback such that `navigator.language` would return `en-US` even when the first item of `navigator.languages` and `Accept-Language` and the ECMA-402 fallback is `en-ZZ`.

(Aside: Comparing the U.S. English cluster above with CLDR unit preference data might appear to indicate that picking a region from that cluster would address the problem to be solved here. Unfortunately, there’s no region in the cluster that resolves both to international standard units and to the ISO week rule per CLDR. Even ignoring the minDays count="4"` aspect of ISO weeks, the unit-wise best candidates of `PH` and `VI` have Sunday in the first column of tabular calendar views.)


## Acknowledgements

The author of this explainer would like to acknowledge Shane Carr, Eemeli Aro, and Tantek Çelik for discussions and ideas that resulted in this write-up and the authors of previous proposals in this space Ben Allen and Romulo Cintra.


## Appendix 1: Expected patches to CLDR

* [Change `minDays count="1"` to `minDays count="4"` for `001`](https://unicode-org.atlassian.net/browse/CLDR-18506). This would be a worthwhile change on its own regardless of the rest of this explainer. (Alternatively, specify `minDays count="4"` for `ZZ`.)
* Add `ZZ` to the English paradigm list that contains `US`. (Even without this explainer, it’s a bit strange that region-unqualified English and Unknown Region-qualified English end up in different English paradigms per CLDR currently.)
* Add `en_ZZ.xml` inheriting from `en.xml` and overriding short date (sub)patterns. For Gregorian, change `M/d/y` to `y-MM-dd`, `M/y` to `y-MM`, and `M/d` to `MM-dd` in patterns.

It's unclear what, if anything, should be done about non-Gregorian `M/d/y`, `M/y`, and `M/d` patterns. To the extent the other calendars are strongly associated with a region, in most cases those regions, too, use the slash as the component separator for short dates with all three of little-ending, big-endian, and middle-endian occurring, so with the possible exception of the Hebrew calendar (dot for little-endian), there aren't punctuation separators that are neither the slash nor the hyphen that would obviously communicate short date endianness for non-Gregorian calendars without diluting the ISO meaning of the hyphen, i.e. big-endian Gregorian (as opposed to big-endian more generally).


## Appendix 2: Ways en-ZZ is expected to differ from en-US

### Currency

**Risk:** `ZZ` currently lists various codes starting with `X` that don't signify normal currencies (precious metals and such) and does not list `USD`. Notably, all currency entries for `ZZ` have `tender="false"`. This might break something the queries for the currency of the region, but when the user isn't actually in the U.S. context, `USD` isn't necessarily right, anyway.


### Time Zones of Locale

**Risk:** Querying for the time zones of the locale won't list the time zones of the United States. While this might break something, it might even be a slight improvement to how time zone pickers are pre-populated for users who aren't actually in the United States. That the `US` region has more than one time zone is probably a mitigating factor, since sites can’t assume the region to map to exactly one time zone.


### Calendar

No change. (Inherit `gregorian` from `001`.)


### Week data

`firstDay day="mon"` (as opposed to `sun`). `minDays count="4"` (as opposed to `1`). No change to weekend (`sat` and `sun`) or `weekOfPreference ordering="weekOfYear"`.


### Hour cycle

Inherit from `001` resulting in `h23` (as opposed to `h12`). (Time separator unchanged, i.e. colon.)


### Datetime formats

`M/d/y`, `M/y`, and `M/d` patterns for Gregorian changed to `y-MM-dd`, `y-MM`, and `MM-dd` per above.


### Measurements

Inherit from `001` (`metric`, `A4` paper).


### Units

Inherit from `001` (changes various units to metric).


### Collation

No change. (Root collation.)


### Person name

No change. (`givenFirst`)


### Text direction

No change. (English is left-to-right regardless of region.)


### Numbers

No change. (`Latn` digits, decimal and thousand separators unchanged, what "billion" means unchanged)
