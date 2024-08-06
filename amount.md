# Amount explainer

Author: [Tantek Çelik](https://tantek.com/)

## User problems to be solved

On the web, users from around the world often see numerical amounts on web pages either in unfamiliar units, 
or with unfamiliar punctuation denoting large or decimal amounts which can either be hard to interpret, or worse, easily misinterpreted. 
If publishers had a mechanism to explicitly denote these numerical amounts and their units if any, 
then browsers could offer a privacy-respecting user-interface to display these amounts to the user according to their local units and punctuation preferences.

## Methodology for approaching & evaluating solutions

Per the priority of constituencies, we should look for solutions that users can trust (like if there is an affordance to convert units, it should be 100% reliable when present), and second, solutions that are more dependable and robust for developers. In particular, we should look for and prefer declarative approaches over imperative approaches per [the Mozilla Web Vision, e.g. for more “robust & predictable fallback behaviors”](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb). Among such solutions, we should see if there have been past proposals, especially by web developers, and see if we can re-use any part of their ideas, semantics, or syntax.
