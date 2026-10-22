# File System Access: scoped, visible and revocable grants

* **Short name:** `file-system-access-scoped-grants`
* **Status:** Draft counter-proposal, for discussion. This explainer does not change Mozilla’s [standards position](https://github.com/mozilla/standards-positions/issues/154) on the File System Access API.
* **Authors:** [Simon Pieters](https://github.com/zcorpan), [Arnaud Bienner](https://github.com/ArnaudBienner), [Harveer Singh](https://github.com/hsingh-moz), [Andrew Sutherland](https://github.com/asutherland), [Sam Lloyd](https://github.com/slloydmoz)
* **Feedback:** https://github.com/WICG/file-system-access/issues/475

## Summary

The [File System Access API](https://wicg.github.io/file-system-access/) lets a website ask the user to pick a file or folder on their device, then read and write it directly. Chromium has shipped it since Chrome 86 (2020). Gecko and WebKit ship only the [Origin Private File System](https://fs.spec.whatwg.org/#origin-private-file-system), which lives in the WHATWG [File System Standard](https://fs.spec.whatwg.org/). Both have published negative positions on access to the user’s own files ([Mozilla](https://github.com/mozilla/standards-positions/issues/154), [WebKit](https://github.com/WebKit/standards-positions/issues/28)).

This explainer proposes changes that would let the use cases work while making each grant:

1. **narrower**: it covers what the task needs, not everything in a folder;

2. **less dangerous**: it cannot be used to plant executables or overwrite files unmarked;

3. **shorter-lived**: one decision does not become standing access by default;

4. **held by the right code**: sites can keep third-party scripts on the page away from it;

5. **visible and revocable**: users can see when it is in use and take it back.

Some of these are things a browser can already do within the spec’s latitude. The rest need normative spec changes, and those are the focus of this proposal.

## 1\. User problems

People want to use web apps to work on their own files and folders. For example, they want to:

* open a project folder in a web-based code editor and save changes back to it;

* keep notes as plain Markdown files in a folder they control;

* back up a local photo folder automatically;

* copy firmware onto a microcontroller that mounts as a drive;

* edit or transcode media files too large to upload or hold in memory.

When a browser doesn’t support this, users are told to switch browsers or are asked to install a native app, which gets unrestricted access to their device or account.

Users also need a grant to mean what they think it means. “This site can work on this folder, for this task” should not quietly become “this site, and any script it loads, can read everything in this folder indefinitely.”

## 2\. Methodology

We evaluate the API against the [Mozilla Web Vision](https://www.mozilla.org/en-US/about/webvision/full/#safetyfornewcapabilities)’s criteria for capabilities that cannot be safely exposed by default:

> * *Value:* Does the capability deliver sufficient benefit to the user to warrant the risk?
> * *Severity:* What level of harm can occur if the capability is misused?
> * *Consent:* Can individuals make an informed decision about the risk?
> * *Transparency:* Is it clear how the capability is being used, and by whom?
> * *Revocability:* What happens if someone changes their mind?

The Web Vision also notes that “permission prompts alone cannot make every capability safe” and that “many people just accept whatever prompts they’re offered \- particularly as they become more common.” So we prefer mitigations that bound the harm structurally over ones that depend on the user reading a dialog.

We also require that a proposal keep the use cases in §1 working, and that more than one engine can implement it.

## 3\. The existing API

The local-file part of the API has three entry points: `showOpenFilePicker()`, `showSaveFilePicker()` and `showDirectoryPicker()`. Each one requires a secure context, transient user activation and a same-origin-with-top-level document. Each returns a *handle*. Handles can be stored in IndexedDB and retrieved on a later visit; a retrieved handle’s permission defaults to `"prompt"`.

It helps to think of the API as a stack of capabilities, each strictly larger than the one below it:

| Layer | Capability | Mechanism | Privacy and security delta |
| :---- | :---- | :---- | :---- |
| L0 | Sandboxed per-origin storage | OPFS (`navigator.storage.getDirectory()`) | None beyond existing storage; shipped in all engines |
| L1 | Read a user-picked file | `showOpenFilePicker()` | Small; `<input type=file>` already does this |
| L2 | Write a user-picked file | `showSaveFilePicker()`, `createWritable()` | Medium; a page can overwrite a user’s file |
| L3 | Read/write a user-picked directory tree | `showDirectoryPicker()` | Large; one grant covers an arbitrarily large subtree |

**Persistence** is an independent property that can apply to L1–L3. The spec [encourages](https://wicg.github.io/file-system-access/#privacy-tracking) user agents “to automatically expire permission grants except for particularly well trusted origins (for example persistent permissions could be limited to ‘installed’ web applications)”. Since [Chrome 122](https://developer.chrome.com/blog/persistent-permissions-for-the-file-system-access-api), Chrome offers an “Allow on every visit” option that grants access until the user revokes it.

## 4\. Limitations of the current design

### 4.1 Security moves from structural to prompt-mediated

Two long-standing properties of the web are that pages cannot reach the host file system unless the user carries something across, and that a compromised site can only damage what its origin already holds. The API doesn’t break these properties so much as convert them. Both become “allowed, if the user accepted a dialog, possibly long ago.” Prompt-mediated security has well-known failure modes: habituation, inconsistent understanding of the stakes, and consent given in one context that governs actions in a very different later one.

### 4.2 The grant is broader than the task

* **Over-collection.** A folder granted for one purpose (“back up your photos”) can be enumerated and read in full: tax documents, scanned IDs, work files that happen to live there. Nothing ties the handle to the purpose the site described.

* **Type filters are cosmetic.** `types` on the file pickers only shapes the picker UI, and `showDirectoryPicker()` has no type option at all. A site has no way to ask for only `.jpg` files in a folder, and no way to be held to it.

* **Hidden and credential files.** Dotfiles, SSH keys, cloud credentials and application profiles are visible to directory enumeration unless the user agent chooses to hide them.

* **Metadata.** Photos and documents carry location, device and timestamp metadata that users rarely know they are sharing.

### 4.3 A compromised origin gets the user’s folder

A site with write access to a directory gives anyone who compromises that site the ability to:

* drop executables, scripts or plausibly named documents into the folder;

* overwrite files the user already trusts and opens by habit, such as build outputs, documents or configuration;

* plant DLLs or other load-order payloads next to existing executables;

* embed payloads in existing macro-capable documents.

The File System Standard runs implementation-defined malware scans when a writable stream is closed; beyond that, the spec’s defences against these are non-normative. It “encourages” Mark-of-the-Web and suggests blocking only `.lnk` and `.local` writes. The [RøB paper](https://www.usenix.org/conference/usenixsecurity23/presentation/oz) (USENIX Security 2023\) demonstrated browser-based ransomware built on this API.

“Compromise” here includes any third-party script running in the origin: analytics, ads, support widgets, transitive dependencies. They run *as* the site, and the API treats them identically. The [polyfill.io](https://sansec.io/research/polyfill-supply-chain-attack) takeover (2024) injected malicious code into more than 100,000 sites that had embedded the script years earlier. The `event-stream` (2018) and `ua-parser-js` (2021) npm compromises show the same pattern for dependencies bundled into a site’s own code. In each case, the operators had done nothing wrong. With directory grants in play, the same class of incident would extend to users’ files.

### 4.4 Persistence turns one decision into standing access

* **Drift.** A site’s ownership, terms or business model can change long after the grant. The grant survives the change, and the user is not told.
* **Asymmetric reversibility.** Granting is one click in context. Revoking means finding a settings page and remembering which site you granted, perhaps years ago.
* **Re-identification.** A directory listing (names, sizes, modification times) is stable per user and survives cookie clearing. The spec’s own [privacy considerations](https://wicg.github.io/file-system-access/#privacy-tracking) acknowledge the tracking risk.
* **Coercion.** “Log into this site for me and give it your Documents folder” becomes something a person can be pressured into.
* **Prompt normalisation.** As more sites use the prompt, it loses its signal value. We’ve seen the same with notifications and geolocation.

### 4.5 Where the concerns concentrate

| Concern | L1 | L2 | L3 | Persistent grants |
| :---- | :---- | :---- | :---- | :---- |
| Over-collection / metadata | Per file | Per file | Subtree | Subtree, recurring |
| Compromised-site blast radius | One file (read) | One file | Subtree | Subtree, on any later visit |
| Fingerprinting | Low | Low | High | High |
| Prompt normalisation | Low | Moderate | High | High |

L1 and L2 carry small versions of most concerns. Directory grants (L3) and persistence are where the structural shift happens, and persistence amplifies everything else.

## 5\. Motivation

* **The concerns are concentrated.** They sit in directory scope, third-party code and persistence. Targeted changes there can address most of the risk without removing the use cases.
* **Much of the needed protection is already in the spec as advice.** Making these protections normative gives users consistent protection across browsers and gives developers one predictable target.
* **There is precedent for a stronger trust gesture.** The Web Vision suggests letting users “extend their browser to provide elevated capabilities to specific sites they trust.” Firefox already gates [Web MIDI and Web Serial](https://hacks.mozilla.org/2026/05/web-serial-support-in-firefox/) behind a site-permission add-on prompt that explains the capability in more detail than a standard permission prompt.

## 6\. Proposed changes

Each item is marked **Spec** (needs a normative change), **UA** (a browser can do it within the current spec), or **Both**.

### A. Narrow what a grant covers

| Change | Where | Notes |
| :---- | :---- | :---- |
| Add `types` to `showDirectoryPicker()` and make it **binding**: enumeration and lookup only return matching entries | Spec | Turns “this folder” into “the relevant files in this folder.” |
| Promote the “too sensitive or dangerous” examples to **requirements**: no home, Desktop, Documents or Downloads roots; no browser profile or system directories | Spec | Low cost to legitimate use. |
| Exclude hidden files and dotfiles from enumeration by default, with a user override | UA | Closes the SSH key, cloud credential and profile classes for most users. |
| Require picker UI to state the capability, e.g. “This site will be able to read and change everything in *Photos*” | Spec | Today the picker can look identical to an ordinary OS open dialog. |
| Keep directory read and write as separate consents; `mode: "readwrite"` should not collapse them into one step for a fresh grant | Spec | Workable for editors: one extra prompt at first save. |

### B. Narrow what a grant can do

| Change | Where | Notes |
| :---- | :---- | :---- |
| Normative write blocklist for executable, script and shortcut formats (`.exe`, `.dll`, `.bat`, `.ps1`, `.vbs`, `.scr`, `.msi`, `.lnk`, `.local`, …) and OS autorun locations | Both | Could be overridable per origin for web IDE workflows. |
| Normative write restrictions for macro-capable document formats | Both | Some office-style apps will need an override. |
| Normative read restrictions for credential-pattern files (`id_rsa`, `*.pem`, `.env`, `credentials`) | Both | Same override consideration. |
| Mark-of-the-Web (or platform equivalent) **required** on files created or modified by the API | Spec | The spec already encourages this for both; making it a requirement (if supported by the platform) keeps overwritten files from losing the protection in some implementations. |
| Rate-limit bulk operations per grant and surface a warning | UA | Reading 10,000 files in 10 seconds is not what a photo editor does. |

### C. Narrow how long a grant lasts

| Change | Where | Notes |
| :---- | :---- | :---- |
| No silent re-grant by default: retrieved handles prompt in each new session | Both | Already allowed by the spec; making it the default normatively gives users the same behaviour across browsers. |
| Persistent grants only after a substantive trust gesture, such as installation or an add-on-style capability prompt | Both | Keeps backup and notes use cases viable on terms the user can understand. |
| Persistent grants are time-bounded and lapse after a period of non-use | Both | Catches the “granted years ago, site later compromised” case. |
| Re-confirm the first use of a grant after sustained inactivity, even within a session | UA | Restores awareness during long-lived sessions. |

### D. Narrow who can exercise a grant

This is the supply-chain axis and, we believe, the highest-leverage spec change.

Scripts within one origin can’t be reliably told apart (see §9), so the boundary has to be the origin. Sites then have two ways to keep untrusted code away from grants:

* Keep the top-level page free of untrusted scripts and put those in cross-origin frames. This works today, but it means trusting every dependency bundled into the page, and many third-party scripts don’t work in a frame.

* Treat the top-level page as untrusted and move file handling to a dedicated origin. This needs the changes below.

| Change | Where | Notes |
| :---- | :---- | :---- |
| **Policy-controlled feature** for the API, with a default allowlist of `'self'`, so a page can disable it for itself and its frames, e.g. `Permissions-Policy: file-system-access=()` | Spec | Standard Permissions Policy. Also lets sites that don’t use the API turn it off. |
| **Delegation to a frame only:** let a page delegate the API to a specific same-site, cross-origin frame while it’s disabled for the page itself | Spec | Permissions Policy prevents this today: a frame inherits “Disabled” whenever the feature is disabled for its parent ([*Define an inherited policy for feature in container at origin*](https://w3c.github.io/webappsec-permissions-policy/#algo-define-inherited-policy-in-container), step 2). Needs a Permissions Policy change, and relaxing the API’s same-origin-with-top-level requirement to same-site. The frame must be in an origin-keyed agent cluster, so the page can’t merge with it through `document.domain`. |

Without the second change, the dedicated origin can run in a top-level window (a popup or tab) that the main page opens and talks to through `postMessage`. That works with the API’s current rules, but a separate window is probably not acceptable UX for most apps.

Either way, handles can’t leak to the main page: `FileSystemHandle` deserialization already fails across origins. But the dedicated origin protects files only if it keeps them to itself. Anything it sends to the main page (`File` objects, contents, transferred streams) is available to every script there. So reading, processing and writing should happen in the dedicated origin, and it should treat messages from the main page as untrusted: no generic “write this path” or “delete” operations, and destructive or bulk changes confirmed by the user in its own UI. Its own scripts become the trusted base, so it needs few dependencies, Subresource Integrity and a strict CSP. The main page can navigate the window or frame to any URL on the dedicated origin, so that origin should host only the file-handling page.

Both the window and the frame option rely on the `Permissions-Policy` header for opting out, which Gecko doesn’t ship yet (it supports the `allow` attribute).

### E. Make grants visible and revocable

| Change | Where | Notes |
| :---- | :---- | :---- |
| Persistent indicator while a grant is active, alongside camera and screen-share indicators | UA | Strong precedent. |
| Per-handle (not just per-origin) revocation from the address bar and site information panel | UA |  |
| Activity log: “in the last week, example.com accessed these files” | UA | Lets users audit without third-party tools. |
| One-click pause without revoking | UA | Like pausing a camera. |

### F. Limit file data exposed

| Change | Where | Notes |
| :---- | :---- | :---- |
| Expose only the metadata the API needs | Spec | Reduces fingerprinting; some apps legitimately need modification times. |
| Optionally strip location and device metadata from images on read | Spec | Has costs for apps that need the metadata; open question. |

## 7\. Usage

For most sites, the developer-facing changes are small. A photo backup app would ask only for what it needs:

```js
// Proposed: `types` on showDirectoryPicker is binding.
const dir = await window.showDirectoryPicker({
  mode: "read",
  types: [{ description: "Photos", accept: { "image/*": [".jpg", ".jpeg", ".heic", ".png"] } }],
});

for await (const [name, entry] of dir) {
  // Only matching image files and directories are returned.
  // Hidden files are excluded by default.
}
```

A site that wants to keep third-party scripts on its pages away from the API would turn the API off on those pages and serve file handling from a dedicated origin (§6.D):

```
Permissions-Policy: file-system-access=()
```

```js
// Works with today's rules, but opens a separate window.
const files = window.open("https://files.example.com/editor");
```

With the proposed Permissions Policy change, the dedicated origin could be a frame instead. How the page would express “delegate, but not to myself” is an open question:

```html
<iframe src="https://files.example.com/editor" allow="file-system-access"></iframe>
```

An editor that needs to write asks for write access at first save, not at pick time:

```js
const dir = await window.showDirectoryPicker({ mode: "read" });
// ...later, on the user's first save:
await dir.requestPermission({ mode: "readwrite" });
```

## 8\. Caveats and open questions

* **These changes reduce risk; they don’t remove it.** Directory access still enlarges the blast radius of a site compromise beyond the origin’s own storage. Fingerprinting from directory structure is reduced, not eliminated. Platform-level effects such as prompt normalisation and lock-in to apps that hold grants can’t be solved by any single browser.

* **Delegation to frames.** Permissions Policy deliberately prevents a page from delegating a feature it has disabled for itself, so the frame option needs agreement there. The API also currently rejects documents that aren’t same origin with the top level. We’d like to understand why (perhaps because permission UI names the top-level origin). Relaxing it only for same-site frames keeps the prompt truthful about the site, and the prompt can also name the frame’s origin, e.g. “files.example.com on example.com”.

* **Off by default.** The policy-controlled feature could default to disabled, so only sites that send a `Permissions-Policy` header enabling it get the API at all. Injected scripts on the many sites that never meant to use the API then couldn’t ask for write access or live directory handles (`<input type=file>` still gives read-only snapshots), and the capability would be declared by whoever controls deployment rather than by script. It doesn’t seal anything by itself: once a site opts in, every script on that origin has the API again. It has costs: Permissions Policy default allowlists can only be `*` or `'self'`, so it needs a framework change; sites that use the API today would need to add the header, and Chromium would need to change too; if the methods are exposed but fail without the header, sites that feature-detect take the broken path; and Permissions Policy has no `<meta>` delivery, so apps on static hosts couldn’t opt in.

* **Detecting whether the API is allowed.** `'showDirectoryPicker' in window` only says the method exists, and the API’s permission queries are per handle, so a site can’t tell before the first picker call whether it will work. Permissions Policy defines `document.permissionsPolicy.allowsFeature("file-system-access")` for this, but it isn’t widely shipped (Gecko has it behind a pref). Shipping it alongside the policy-controlled feature would help sites whether or not the feature is off by default.

* **Overrides.** Blocklists for executables and macro documents will need a per-origin override for legitimate tools. That override then becomes the thing to protect.

* **Persistence and trust gestures.** We need to define what counts as a substantive trust gesture, and how it interacts with installed web apps, which are supported differently across engines.

* **Compatibility.** Binding type filters and dotfile exclusion are observable to sites. Apps written against Chromium’s current behaviour would need to change, and so would Chromium.

* **Overlapping grants across origins.** If the user grants the same directory, or nested directories, to two origins, those origins can communicate through files. The prompt could list other origins that already have access to the directory, a parent or a child of it, and offer to revoke their grants. A user agent that persists grants could prompt again when that list has changed. Sharing a folder between apps can be legitimate (a notes folder used by an editor and a backup app), so we lean towards warning over forced revocation.

* **Tracking files that survive clearing site data.** A site can write an identifier into a granted folder. Clearing site data removes stored handles, so the site can read the identifier again only if the user picks the folder again, similar to re-uploading a file the site gave them. Deleting the site’s files automatically is too risky, since the user may care about them. With an activity log (§6.E), clearing site data could list the files the site created and let the user choose which to delete. Fingerprinting from directory listings survives clearing regardless.

## 9\. Ideas considered and set aside

* **Only allow writing to files the site created.** This would break editors and anything else that changes user content.

* **Expire grants on “origin change” signals** (ownership transfer, terms updates). These are attractive, but not reliably detectable.

* **Per-site allowlists maintained by the browser.** Not a coherent platform policy.

* **Per-script gating and handle sealing.** Letting a site name which scripts may call the API, with handles checking the caller, needs per-script attribution within one origin, and that can’t be made robust. Other scripts can `import()` an allowed module and call its exports, or read an allowed script’s nonce through `element.nonce` and insert a script with it. Bound functions don’t appear on the stack, so a script can get an allowed script to call the API on its behalf. Shared built-ins can be patched so the allowed script’s own reads and writes are redirected. Moving file handling to a dedicated origin (§6.D) uses the origin boundary instead.

## 10\. Destination

Spec changes would be proposed to the [WICG File System Access](https://github.com/WICG/file-system-access) draft. Changes that touch handle semantics (binding filters on enumeration) may also need changes in the [WHATWG File System Standard](https://github.com/whatwg/fs), and delegating the API to a frame only needs a change in [Permissions Policy](https://github.com/w3c/webappsec-permissions-policy). UA-level items are offered as recommendations for the spec’s privacy and security considerations.

If multi-vendor interest materializes, the API should move out of WICG, either to the [W3C Web Applications Working Group](https://www.w3.org/groups/wg/webapps/) or to the WHATWG, where the File System Standard already defines handles and writable streams.

## References

* [File System Access (WICG)](https://wicg.github.io/file-system-access/) and its [privacy](https://wicg.github.io/file-system-access/#privacy-considerations) and [security](https://wicg.github.io/file-system-access/#security-considerations) considerations

* [File System Standard (WHATWG)](https://fs.spec.whatwg.org/)

* [mozilla/standards-positions\#154](https://github.com/mozilla/standards-positions/issues/154), [WebKit/standards-positions\#28](https://github.com/WebKit/standards-positions/issues/28)

* [Persistent permissions for the File System Access API](https://developer.chrome.com/blog/persistent-permissions-for-the-file-system-access-api) (Chrome for Developers)

* Oz et al., [RøB: Ransomware over Modern Web Browsers](https://www.usenix.org/conference/usenixsecurity23/presentation/oz), USENIX Security 2023

* Sansec, [Polyfill supply chain attack hits 100K+ sites](https://sansec.io/research/polyfill-supply-chain-attack), 2024

* [Mozilla Web Vision: Safety for New Capabilities](https://www.mozilla.org/en-US/about/webvision/full/#safetyfornewcapabilities)

* [Announcing Web Serial support in Firefox](https://hacks.mozilla.org/2026/05/web-serial-support-in-firefox/) (add-on gating precedent)


