# Explainer: trusted-or-sanitized-html

Author: [Frederik Braun](mailto:fbraun@mozilla.com) ([@mozfreddyb](https://github.com/mozfreddyb))

Last Update: Jun 16, 2026

# Problem

Despite various prior attempts at mitigation, Cross-Site Scripting (XSS) has remained as [one of the top 3 most-reported security issues for a full decade](https://cwe.mitre.org/top25/archive/2025/2025_cwe_top25.html).

# Methodology for Approaching & Evaluating Solutions

We believe that simplified deployment will increase the usage of XSS protections in the real world. In fact, our research on HTTPS adoption has shown that wide uptake in security features requires simple opt-in mechanisms. As an example, “Upgrade Insecure Requests” steered browsers to `https`  URLs despite HTML code pointing to `http`, essentially doing all the necessary work to make a website fully encrypted with just a small header change \- a change that can be carried out without rewriting any of the existing code.  
Following the model of these opt-in mechanisms we believe that an implicit, easy to use HTML sanitizer would increase the adoption of client-side XSS prevention features and help users address one of the most reported vulnerabilities. Consequently, it makes it [easier for anyone to publish on the web \- safely](https://www.mozilla.org/en-US/about/webvision/#:~:text=Make%20it%20easy%20for%20anyone%20to%20publish%20on%20the%20Web).

# Existing features that attempted to solve the problem

* [Content Security Policy](https://www.w3.org/TR/CSP/)
* [Trusted Types](https://w3c.github.io/trusted-types/dist/spec/)   
* [Sanitizer API](https://html.spec.whatwg.org/#html-sanitization)

The [Trusted Types](https://w3c.github.io/trusted-types/dist/spec/) specification is newly available in all browser engines and already helps enforce control over potentially malicious HTML input. The feature is enabled by sending a Content-Security-Policy header with the `require-trusted-types-for 'script';`  directive.

The [Sanitizer API](https://html.spec.whatwg.org/#html-sanitization) \- which has already shipped in Firefox and Chrome \- provides a counterpart to Trusted Types: It defines simple, safe-by-default HTML insertion APIs (e.g. `Element.setHTML()`), which the browser guarantees to secure. The API accepts optional configuration parameters which enable developers to build their own list of allowed elements and attributes. Two presets already exist: First, a “default” configuration which removes XSS risks as well as exfiltration and UI redressing issues. Second, a “baseline” that only removes script elements and event handler attributes.  The baseline is always in effect and the Sanitizer can not be configured to allow XSS.

# Flaws or limitations in existing features

CSP is widely considered hard to retrofit into existing applications as it prescribes a specific application architecture and strict oversight of where scripts are coming from and what actions they perform (e.g., [about 90% of the deployed policies on the web do not protect against XSS](https://almanac.httparchive.org/en/2024/security#content-security-policy)).

Under Trusted Types, every use of risky APIs  (e.g, assigning `innerHTML`) will throw a `TypeError` unless input has been checked and transformed into an e.g., `TrustedHTML` type. However, these types can only be created by implementing a [TrustedTypePolicy](https://developer.mozilla.org/en-US/docs/Web/API/TrustedTypePolicy), which requires custom code and security expertise. Furthermore, Trusted Types, in itself only provides the input control, but no safety. The HTML sanitization needs to be written by the developer or passed towards a third-party library.  
Lastly, Trusted Type is agnostic to the parsing context of the specific element. A type that is considered trusted for one specific element’s innerHTML steps will behave differently. As an example, the browser will only decide [how to parse an input](https://html.spec.whatwg.org/#fragment-parsing-algorithm-steps) after the Trusted Types steps have been run.

While the Sanitizer API provides safety, it lacks the means to enforce document-wide control. Theoretically, existing code could be easily secured by replacing `innerHTML` assignments with `setHTML()`. However, assurance about written source code is easily violated by third-party JavaScript \- which is neither written by first-party developers nor guaranteed to be stable.

# Our Proposal

We believe that a combination of Trusted Types and the Sanitizer API can provide an easy-to-use switch, which solves XSS for existing documents. We can build it by relying on the safety from the Sanitizer API as well as the control functions provided by the Trusted Types API.  
That way, developers are neither required to write policy code, nor are they required to think about what HTML elements and attributes can be misused by a potential attacker.

We suggest that an *implicit* TrustedTypePolicy, which is part of the browser, could provide this safe combination: Rather than implementing and allowing a custom policy using the `trusted-types` directive in CSP, developers would allow a policy under the name `sanitize-html`, which the browser recognizes as a builtin default sanitizer configuration. The slightly weaker baseline would be represented by `sanitize-html-baseline`.   
When one of these policies is enabled, the browser knows to directly sanitize all HTML that is flowing into potentially risky APIs (like `innerHTML`) rather than throwing a `TypeError`. 

## Examples

With one of the proposed keywords set, potentially malicious input will not reach sensitive HTML parsing sinks. Given:

`Content-Security-Policy: require-trusted-types-for 'script'; trusted-types 'sanitize-html';`

```javascript
// Today, under require-trusted-types-for 'script': throws a TypeError.
// With this proposal: parses, sanitizes, inserts.
el.innerHTML = '<p onclick="evil()">Hello</p><form><input name="q"></form>';
// Resulting DOM: <p>Hello</p>
// (onclick stripped; form and input are not in the safe default configuration)

const doc = new DOMParser().parseFromString(userHtml, "text/html");
// doc is sanitized; equivalent to Document.parseHTML(userHtml)

```

With `'sanitize-html-baseline'`, the same assignment keeps the markup structure and removes only XSS vectors:

```javascript
el.innerHTML = '<p onclick="evil()">Hello</p><form><input name="q"></form>';
// Resulting DOM: <p>Hello</p><form><input name="q"></form>
// (only onclick stripped; the baseline configuration removes nothing else here)
```

# Design Considerations

## Compatibility & Deployment

This proposal does not address the creation of `TrustedScript` or `TrustedScriptURL` objects, as they cannot be vetted or sanitized by the browser. Pages that load additional scripts from JavaScript will also need to write their own policy code.

Similarly, existing code may be running `setHTMLUnsafe` (and others) with input that is *known* to be safe, because the content is hardcoded or fetched from a trusted entity). Exercising controls over known/accepted usage of something that is only theoretically unsafe can result in extra work to ensure compatibility. We consider this out of scope for the same reason as above: Pages can write minimal policy code that wraps code which is known to be safe into an explicit `TrustedHTML` type, thus making the assumptions explicit rather than implicit. 

## Customization

Sites that need custom sanitization can continue to use Trusted Types policies. Explicit Trusted Types behavior takes precedence over implicit sanitization, so applications can use the header as a default safety net while handling special cases in policy code.

## Future Work: Parser-Integrated Sanitization

Pages that opt in and still call `document.write()` or `document.writeln()` without a `TrustedHTML` type will throw, exactly as under regular Trusted Types enforcement, until sanitization is integrated into the HTML parser \- work we expect to happen in the near term. Until then, such call sites need a `default` policy or a code change. Parser-integrated sanitization would complete the model by bringing `document.write()`, `document.writeln()`, and streaming parsers into scope.

Future parser-integrated sanitization would allow `document.write()` and streaming parsing APIs to be covered, and would eliminate the pre-sanitization side-effect window. This proposal deliberately excludes those cases until such integration exists.

## Open Questions

* **Reporting.** Today, a plain string reaching a `TrustedHTML` sink [reports a violation](https://w3c.github.io/trusted-types/dist/spec/#should-block-sink-type-mismatch) before throwing. This proposal *suppresses* that report for implicitly sanitized assignments: the operation succeeds by design, so there is no violation to report. We are aware of the trade-off and would be happy to discuss it \- a developer rolling out the keyword may want to observe exactly where their markup is being changed, and a dedicated reporting mechanism for sanitization events might serve that need better than overloading the existing violation report. (Report-only policies are unaffected either way: they keep reporting and never sanitize.)

## Security & Privacy Considerations

This proposal intends to improve web security at scale, by introducing additional opt-in security controls.  Web pages that do not use Trusted Types will receive strictly improved security when opting into this mechanism. Web pages that *do* use Trusted Types with very strict input handling should ensure that opting in to this new implicit sanitization does not regress their security model. There are no known privacy implications. No additional data is collected, stored or sent.

## Outline of required specification changes

The specification change will result in the following steps roughly:   
Currently, a sensitive sink[^1], calls into Trusted Type to decide if a type is a) already present, b) required, c) created on the fly. If a type is both required and not available (through manual creation or implicit), the Trusted Type algorithm throws `TypeError` and reports a violation.  
With this change, we allow the browser to sanitize automatically, but at the sink level rather than within Trusted Types. This allows the browser to parse and sanitize the provided input according to the specific context element \- fixing a key shortcoming of the Trusted Types specification.

Structurally, the proposal touches three layers: CSP provides the opt-in, Trusted Types decides *whether* implicit sanitization applies, and HTML performs the sanitization at each individual sink. The sanitization deliberately happens at the sinks \- not in the Trusted Types enforcement machinery \- because sanitization is context-dependent: the same markup parses differently depending on the context node, and only the sink knows that context.

### CSP: an implicit sanitization mode

The new keywords set a per-document *implicit sanitization mode*: `strict` (via `'sanitize-html'`), `baseline` (via `'sanitize-html-baseline'`), or none. The modes map onto the Sanitizer configurations that the HTML standard already defines: `default` uses the [built-in safe default configuration](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#built-in-safe-default-configuration) (as in `new Sanitizer()`), `baseline` uses the [built-in safe baseline configuration](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#built-in-safe-baseline-configuration) (as in `new Sanitizer({})`), which allows all elements and attributes except those that can lead to script execution.

### Trusted Types: pass the string through instead of throwing

The first step for assigning to `innerHTML`, is the [Get Trusted Type compliant string](https://w3c.github.io/trusted-types/dist/spec/#get-trusted-type-compliant-string-algorithm) algorithm. This algorithm runs unchanged, such that the special [Trusted Type policy `default`](https://w3c.github.io/trusted-types/dist/spec/#default-policy-hdr)  keeps precedence: if it exists and its `createHTML` callback produces a `TrustedHTML`, that value is used and no implicit sanitization takes place.

The only behavioral change: when enforcement would otherwise *throw* for a sink expecting `TrustedHTML` (no `default` policy, or no `createHTML` callback) and an implicit sanitization mode is set, the algorithm would not throw. The string passes through to the sink, marked as *pending sanitization*.

Trusted Types decides whether implicit sanitization applies at the point where enforcement would otherwise fail. If the expected type is `TrustedHTML` and an implicit sanitization mode is active, the string reaches the sink marked for sanitization instead of throwing (cf. Appendix / Implementation sketch)

### HTML: sanitize at the sink, in context

Each sink sanitizes at the point where parsed-but-not-yet-inserted nodes exist in their proper context. Concretely, the sinks invoke the existing [set and filter HTML](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#set-and-filter-html) and [sanitize](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#sanitize) algorithms (avoiding additional re-parsing):

* `innerHTML`, `outerHTML` and `insertAdjacentHTML()` run the *set and filter HTML* steps with the mode's sanitizer: the fragment is parsed with the proper context element, sanitized, and the resulting *nodes* are inserted. An `innerHTML` assignment effectively behaves as `setHTML()` (`default`) or as `setHTML()` with the baseline configuration (`baseline`).  
* `DOMParser.parseFromString()` and `Document.parseHTMLUnsafe()` sanitize the resulting document before returning \- equivalent to [`Document.parseHTML()`](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#dom-parsehtml). (These documents are inert; sanitizing them post-parse is free of side effects.)  
* `Range.createContextualFragment()` sanitizes the fragment in its context before returning it. Its quirk of un-marking scripts as "already started" becomes irrelevant: script elements are removed before the fragment is ever returned.  
* `setHTMLUnsafe()`: with `baseline`, the supplied configuration additionally has [remove unsafe](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#remove-unsafe) applied, guaranteeing that supplied configurations cannot reintroduce XSS; with `default`, it behaves as `setHTML()`.  
* `setHTML()` is already safe by construction; whether `default` should additionally constrain explicit custom configurations is an Open Question (see below).  
* `document.write()` and `document.writeln()` are **not yet** subject to implicit sanitization and retain today's enforcement behavior: without a `TrustedHTML`, they throw. They do not parse into a fragment and there’s currently no way to sanitize during parsing. See *Future Work* for the path to covering them.

# Incubation/Standardization destination

This work should be incorporated  into the WHATWG DOM specification, similar to how [Trusted Types was already integrated](https://github.com/whatwg/dom/pull/1268). Additional changes in WHATWG HTML and the W3C Trusted Types Specification will be necessary.

# Appendix

## Implementation Sketch

Trusted Types already answers CSP-dependent questions by *pulling*: its algorithms receive a global object and inspect the global's CSP list directly. [Does sink type require trusted types?](https://w3c.github.io/trusted-types/dist/spec/#does-sink-type-require-trusted-types) does so for the `require-trusted-types-for` directive, and [Should Trusted Type policy creation be blocked by Content Security Policy?](https://w3c.github.io/trusted-types/dist/spec/#should-block-create-policy) does so for the `trusted-types` directive itself \- the directive our keywords live in. Determining the implicit sanitization mode follows this established pattern: a new sibling algorithm, *get the implicit sanitization mode*, given a global, iterates the global's CSP list and returns `default`, `baseline` or none, considering only policies with an `"enforce"` disposition. (Report-only deployments thereby keep today's behavior exactly: violation reports, no behavior change, no sanitization.)

The flag would be passed in the Trusted Types algorithm**:** [Get Trusted Type compliant string](https://w3c.github.io/trusted-types/dist/spec/#get-trusted-type-compliant-string-algorithm) consults the mode at the exact step where it throws today (the default policy produced nothing, and an enforced policy blocks). If the expected type is `TrustedHTML` and a mode is set, it returns the input string together with the mode instead of throwing; all other return paths carry no mode. Sinks destructure the result and run their sanitization step when a mode is present.

The mode is consulted before the [violation reporting step](https://w3c.github.io/trusted-types/dist/spec/#should-block-sink-type-mismatch): an implicitly sanitized assignment succeeds by design and generates no violation report. 

## Alternatives Considered

### Different layering, outside of “Get Trusted Types compliant string”

The sink reacts to the enforcement failure. *Get Trusted Type compliant string stays* untouched. Each sink (or one shared wrapper algorithm in the HTML specification) invokes it and, if it throws, consults get the implicit sanitization mode itself, sanitizing instead of rethrowing. This is the most literal reading of "change the sinks, not the machinery", but it has a correctness flaw: the algorithm also rethrows exceptions raised by a default policy's createHTML callback, and a sink catching a `TypeError` cannot distinguish an enforcement failure from a throwing callback. A `default` policy that deliberately rejects a value by throwing would be silently converted into implicit sanitization.

### Central integration in the Trusted Types machinery

An earlier draft of this proposal performed the sanitization inside Trusted Types' *Get Trusted Type compliant string* algorithm: parse the string into a fragment, sanitize it, serialize it back, and wrap the result in a `TrustedHTML`. This had the appeal of a single, central specification change, but three problems convinced us of the per-sink design:

* **No context.** Sanitization is context-dependent \- the same string parses differently depending on the context node \- and the central algorithm has no reliable access to that context.  
* **Mutation XSS.** The serialize/re-parse round trip re-opens the [mutation XSS](https://html.spec.whatwg.org/multipage/dynamic-markup-insertion.html#sanitizer-security-mxss) class that the Sanitizer API specifically avoids.  
* **Wrong layer.** Some affected APIs (e.g. the `setHTML()`/`setHTMLUnsafe()` interactions) never pass through the Trusted Types machinery at all.

### A `TrustedHTMLFragment` type

To avoid the serialize/re-parse round trip within the central design, a new type (working title `TrustedHTMLFragment`) could carry the parsed-and-sanitized fragment instead of a string \- or, alternatively, the backing data of `TrustedHTML` could be changed in a compatible way so that implementations may carry a fragment internally. This could also solve performance questions in Trusted Types more generally. With the per-sink design, however, no Trusted Type object needs to be minted on the implicit path at all, so neither change is required for this proposal. This may be worth revisiting as an independent Trusted Types improvement.

[^1]: `innerHTML`, `outerHTML`, `insertAdjacentHTML()`, `document.write()`, `document.writeln()`, `setHTMLUnsafe()`, `DOMParser.parseFromString()`, `Range.createContextualFragment()`, `Document.parseHTMLUnsafe()`.
