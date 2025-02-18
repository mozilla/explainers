# WAICT (Web Application Integrity, Consistency, and Transparency)

## 1. What Are We Doing?
We aim to **raise the security and privacy bar for web applications** by ensuring three key properties:

1. **Integrity**: Prevent any unauthorized modification of site code—even if served by a third-party CDN.
2. **Consistency**: Give users confidence that they receive the *same* code everyone else does (i.e., no user-specific malicious versions).
3. **Transparency**: Ensures stronger security for the Consistency mechanism. Encourage append-only logs that record published code versions, deterring hidden or short-lived changes and helping external audits.

Our goal is a **standardized, browser-based mechanism** that sites can easily opt into—without forcing users to install extensions. While we propose high-level strategy and a technical proposal here, **technical details remain open for discussion** among browser vendors, web application providers, and standards bodies.

---

## 2. Why Are We Doing This?
### Security-Critical Applications
Webmail, secure messaging, banking, and other sensitive sites face *strong* threats:
- **Insider Threats**: Malicious insiders, internal bugs, or selective content modifications.
- **Hosting Compromises**: Attackers or misconfigurations can serve altered code.

### User Trust
This is particularly vital for end-to-end encryption: the page’s integrity is the most important part of the security chain. (because it's encrypted everywhere else, the endpoints are the weak link, so ensuring security is strong there is paramount)

### Extensions & Explicit Consent
Currently, only specialized browser extensions (like CodeVerify) can provide stronger assurances. But extensions require extra user steps, limiting adoption. Our approach makes these protections native and more widely available.

---

## 3. How Are We Doing This?
1. **Leverage & Extend SRI**
   - Instead of a per-resource hash, we propose a **manifest-based** approach listing all subresources (script, style, etc.) with, typically, a single signature covering the manifest to remove performance problems.

2. **HTTP Header (possibly an extension of CSP)**
   - Introduce an “integrity-manifest” directive to specify:
     - **Where** the browser fetches the manifest (`integrity-manifest-src`).
     - **How** strictly to enforce integrity (`integrity-enforcement-level`).
   - If necessary raise UI or block the page based on the enforcement level.

3. **Add Transparency & Consistency**
   - **Signed Manifests & Logs**: Sites can embed cryptographic proof of logging each manifest into an append-only record.
   - **Proof of Inclusion**: Browsers verify that they trust the proof of inclusion but do not need to look into the content. The auditability prevents stealth “one-off” versions.
   - **Downgrade Prevention**: Once a domain is known to enforce these policies, it can request browsers to not accept weaker modes for a specific amount of time (not exactly like, but similar to HSTS).

4. **Prototype & Standardize**
   - We’ll refine prototypes through collaboration with major actors (e.g., WebAppSec at W3C) and consider parallel IETF work for the logging/transparency pieces.
   - **Implementation details** though we want to align on the high level information to raise to the user (if any), this should likely remain implementation specific on the browser side.

---

## 4. Why Integrity and Transparency Together?
- **Short Attack Windows**: Without transparency, a malicious version can appear briefly and vanish with minimal trace. It could achieve local consistency even if checked through multiple network paths.
- **Stronger Deterrence**: A public log, plus manifest-based integrity, makes *every* release visible. Hiding malicious changes becomes much riskier.
- **Efficient Cohesion**: Designing them simultaneously avoids redundant or conflicting mechanisms and simplifies adoption for large, dynamic sites.

---

## 5. Why We Think We Can Actually Do This
- **Real Precedent: CodeVerify**
  - Meta built an extension verifying scripts for WhatsApp, Instagram, and Messenger. It works in production, used by actual users, proving feasibility-though it remains optional and partial.
- **Scalability for Large Sites**
  - Our manifest approach is explicitly designed for high-traffic, multi-CDN services and A/B testing.
- **Browser & Site Interest**
  - Multiple vendors have indicated willingness to explore and prototype stronger security.
- **Incremental Adoption**
  - Non-participating sites are unaffected. Early adopters can start with basic script integrity and then expand to styles, WASM, or further subresources. They can also choose to deploy different enforcement levels and/or enable the downgrade protection gradually.

---

## 6. Why Aren’t SRI or CodeVerify Sufficient on Their Own?
1. **Subresource Integrity (SRI)**
   - **Limitations**: Requires distinct hashes colocated for every resource; large sites find this unwieldy. It also *does not* provide transparency or consistency assurances. It is restricted to scripts at this time. SRI also provides no inclusion or exclusion benefits on the set of scripts. This might be included in transparency, as it governs the addition or removal of scripts for individual users or page loads. Similarily, import-map-integrity only covers ES modules and not packages and Sig-based SRI is too costly for large sets of resources and doesn't cover Consistency...
2. **CodeVerify Extension**
   - **Limitations**: Relies on user installation, cannot verify all resource types, and remains out-of-band from standard browser features. Browser Extension APIs are insufficent for strong protections.

By integrating such ideas *natively*—with a manifest-based approach plus transparency logs—we bring increased default security to all users without requiring an extension.

---

## 7. Who Needs It, and Who Will Deploy It?
- **Needs It**
  - **High-Security Apps**: Secure messaging, financial portals, healthcare sites.
  - **Smaller Sites**: Quickly benefit from the same protections with minimal overhead.
- **Willing to Implement**
  - **Browser Vendors**: Indicated interest in providing a standard solution to solve the strategic security objective. Firefox is invested in the topic and would ideally prefer to implement a standard.
  - **Large Content Providers**: Entities serving traffic eg. Cloudflare or Critical sites like Meta’s Facebook, WhatsApp, Instagram sites.

---

## 8. Conclusion and Next Steps
We propose a **two-phase** approach:
1. **Integrity**: Extend SRI with a manifest-based signature approach and a header to announce the integrity configuration.
2. **Consistency & Transparency**: Integrate cryptographic proofs and public logging to deter targeted attacks and enable long-term audits.

Our next steps include **prototyping** these mechanisms, collaborating across browsers and large-scale websites, and preparing a draft for standardization in W3C WebAppSec and possibly IETF. We want to **align on the high-level strategic idea** that browsers can natively enforce integrity, consistency, and transparency—while acknowledging that **the specifics of this proposal remain open to technical debate**.

By designing integrity and transparency together, we can make the web more trustworthy, ensuring that all users can rely on security-critical sites to deliver the code they promise, consistently and visibly.

---

## 9. More details

You can find more details in [this document](https://docs.google.com/document/d/16-cvBkWYrKlZHXkWRFvKGEifdcMthUfv-LxIbg6bx2o).
