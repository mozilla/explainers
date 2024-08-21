# Mozilla Explainers

This repo is for the development of explainers from Mozilla contributors. 
Mozilla Explainers can start with as little as a sentence summarizing a user-relevant problem 
and how a new technology could help solve it.
The [W3C TAG Explainer](https://tag.w3.org/explainers/) provides a basic outline, examples, and tips for developing explainers.

In general we expect to focus on developing explainers that:
* describe a user-relevant problem as prioritized by the [Mozilla Web Vision](https://www.mozilla.org/en-US/about/webvision/full/) (Web Vision)
* and propose a solution aligned with the Web Vision
* or a counterproposal that is more aligned with the Web Vision than existing technologies or proposals

## Current Explainers
* [Privacy-Preserving Attribution Measurement API](https://github.com/mozilla/explainers/tree/main/ppa-experiment) ([ppa-experiment issues](https://github.com/mozilla/explainers/issues?q=is%3Aissue+is%3Aopen+%5Bppa-experiment%5D))
* [EyeDropper as &lt;input>](https://github.com/mozilla/explainers/blob/main/eyedropper-input.md)
* [Amount](https://github.com/mozilla/explainers/blob/main/amount.md)
* ...

## Feedback on Explainers
We strongly prefer external feedback only on explainers that are in their own repo (see [Explainer file progress](#explainer-file-progress)).

If you see something in a Mozilla Explainer that you want to suggest changing or fixing, please file an issue describing the problem and suggested resolution, not a pull request.  
Exception: minor non-substantive or typographical changes may be submitted as PRs.

## Migrated Explainers
Explainers migrated to an incubation group or standards working group.
* none currently.

## Archived Explainers
Explainers we are no longer pursuing, have been superseded, or have been incorporated into another explainer.
* none currently.

## How to add an Explainer
1. write up a minimum viable (or more) explainer
2. get reviews and approvals from your senior standards person (ask [@tantek](https://github.com/tantek) for who) and the CTO office (ask [@martinthomson](https://github.com/martinthomson) for who)
3. convert your draft explainer to markdown
4. submit a pull request that adds the file to this repository and links to it from the [Current Explainers](#current-explainers) section above

## Minimum viable explainer
Start your explainer with at least:

0. a short name (for the directory that will hold your explainer)
1. user problem(s) to be solved with a new/improved standard, at least a one sentence description

If your explainer is a counter-proposal to an existing public explainer or group proposal, 
your minimum explainer should include enough additional sections to provide a useful contribution 
to the existing conversation around the problem(s) being solved, and approaches already being explored.
For example, if someone else has proposed an API to solve the problem(s) your explainer describes, 
your minimum explainer should include at least sections 1-5 (see [below](#explainer-sections)), 
especially existing proposal flaws, and why/how we can do better.

## Explainer sections
Expand your explainer with the following sections, in order:

2. methodology for approaching & evaluating solutions (e.g. cite & quote from the Web Vision)
3. prior/existing features and/or proposals (if any) that attempt to solve the problem(s)
4. flaws or limitations in existing features/proposals that prevent solving the problem(s)
5. motivation for this explainer, why we think we can do better than the status quo or other proposals
6. outline of a proposed solution
7. usage, examples, sample code and a prose summary of how it clearly solves the problem(s)
8. caveats, shortcomings, and other drawbacks of design choices, both current and any prior iterations
9. draft specification
10. incubation and/or standardization destination, this can be:
 * an existing standards-track spec (if any) to incorporate the proposed solution as a feature,
 * an incubation destination (e.g. WICG, WHATWG stage 0, or TC39 stage 0),
 * or a standards working group, e.g. CSS WG (which incubates internally) or a WHATWG work stream if applicable

## Explainer file progress
This repo is for developing explainer files until ready for external feedback, 
or archival if we decide an explainer is no longer worth pursuing.

For new minimum explainers (those missing any of sections 2 through 6),
we expect that those sections are added within 2-4 weeks of
publishing your minimum explainer, using pull requests.

When ready to solicit external feedback on an explainer:
* create a new repo for the explainer (same short-name if possible), 
* move the explainer file into that new repo (with history)
* add a link to the new repo in the Current Explainers section

Alternatively, archive an explainer file if:
* we decide the problem isn’t worth solving, or is better solved some other way, and document why in the explainer
* we write a new explainer with a different problem framing that supersedes the explainer, and link from it to the new explainer
* an explainer by another organization incorporates our explainer, and we link from it to the external explainer

## Explainer repo progress
Iterate on an explainer repo, improving the explainer 
until there is broader interest to migrate it to an incubation or standardization group, 
or alternatively archive it for the same reasons as above.

Iterate by:
* incorporating feedback
* prototyping the explainer solution to test its efficacy
* working with others to broaden interest

When there is consensus on an incubation or standardization destination for the explainer, 
follow the destination’s process for migrating into it.

## References
Useful reading before/while writing an Explainer:
* https://tag.w3.org/explainers/ — The W3C TAG’s Explainers Explainer
* https://www.mozilla.org/en-US/about/webvision/full/ — Mozilla’s vision for the evolution of the Web

Real world examples of explainers from other organizations (alphabetical by URL) — check these for similar problems being solved, opportunities to collaborate rather than create a new explainer:
* https://github.com/explainers-by-googlers/ 
* https://github.com/MicrosoftEdge/MSEdgeExplainers/
* https://github.com/WebKit/explainers
