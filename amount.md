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
