# Amount explainer

Author: [Tantek Çelik](https://tantek.com/)

## User problems to be solved

On the web, users from around the world often see numerical amounts on web pages either in unfamiliar units, 
or with unfamiliar punctuation denoting large or decimal amounts which can either be hard to interpret, or worse, easily misinterpreted. 
If publishers had a mechanism to explicitly denote these numerical amounts and their units if any, 
then browsers could offer a privacy-respecting user-interface to display these amounts to the user according to their local units and punctuation preferences.

## Methodology for approaching & evaluating solutions

Per the priority of constituencies, we should first look for solutions that users can trust (like if there is an affordance to convert units, it should be 100% reliable when present) and, second, solutions that are more dependable and robust for developers. In particular, we should look for and prefer [declarative approaches](https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb) over imperative approaches.
