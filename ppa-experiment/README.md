# Experiment: Privacy-Preserving Attribution Measurement API

Mozilla is working with Meta and other actors on defining an in-browser attribution API. The purpose of this API is to provide a privacy-first design for advertising companies to be able to measure how advertising drives conversions. That is, answering the question of whether advertising effectively achieves its goals, such as increased sales.

A full version of an in-browser attribution API will offer strong privacy protections, while providing considerable flexibility in how to measure ad performance. Our long term goal is a standardized attribution solution. We believe that a good attribution system will give advertising businesses a real alternative to more objectionable practices, like tracking, which should allow browsers to further restrict those practices.

To inform our work toward a cross-browser standard, Mozilla is looking to trial a lightweight version of a [proposal](https://docs.google.com/document/d/1QMHkAQ4JiuJkNcyGjAkOikPKNXAzNbQKILqgvSNIAKw/edit) we’re working on with Meta and collaborators from the Private Advertising Technology group at the W3C.

For advertisers, the trial version will be considerably less capable and flexible than a full solution. We will not similarly compromise on privacy: privacy protection for the trial should be as strong or stronger than the complete solution. Measurement options will be limited to a simple aggregate count of conversions, with differential privacy noise added.

## End-User Benefit

Users largely benefit indirectly from the use of this API. That’s a hard fact, but an important one.

Any benefit people derive from this feature is indirect. The sites they visit are often supported by advertising. Making advertising better makes it possible for more sites to function using the support that advertising provides.

Our view is that the costs that people incur as a result of supporting attribution is small. Those costs are:

- CPU, network, and battery costs for generating and submitting reports. Here, this cost is negligible, particularly relative to what sites are already able to use. This design could replace some of those costs, which might lead to improvements in some cases.
- Privacy loss from use of their information. Attribution information will be aggregated and will include noise that protects the contribution that each person makes. This design is structured so that advertisers learn about what many people do as a group, not what any single person does.

In comparison, the indirect benefits are likely to be significant:

- The value that an advertiser gains from attribution is enormous. Attribution can be the difference between an advertising-dependent business surviving or failing. Advertising-supported content and services can be more equitable than alternative funding models.
- If advertisers do not need to track people for attribution purposes, it makes it easier for us to identify and stop tracking.

The first point here is hard to measure objectively, but we have at least one example to draw on. Meta famously [reported USD10 billion of losses](https://www.businessinsider.com/facebook-blames-apple-10-billion-loss-ad-privacy-warning-2022-2?op=1) as a result of Apple’s Ad Tracking Transparency feature, which resulted in them being unable to perform attribution for a sizable portion of iPhone users.

Our assessment is that these benefits justify the costs.

## Experiment

The purpose of this trial is to validate that such a system is capable of providing useful information. Advertisers are accustomed to measurement systems that produce highly individualized information, but which have awful privacy characteristics. A system that produces noisy aggregates is a vast privacy improvement, but one that will require some adjustment.

We are deliberately choosing a conservative privacy posture for the experiment to observe the extent to which that affects the usefulness of results.

We also hope to gain some experience with deployment, to inform design choices for a final and more complete solution.

This experiment will be a live trial that runs as an origin trial. That is, only sites that are opted in to the experiment will be able to access the API.

## Operation

Attribution measurement involves measuring actions that occur on different sites. An advertisement is shown on one site, with the conversion (such as a purchase) happening on another site.

In our trial we seek to count the number of attributed conversions. That is, the number of conversions that follow an ad impression for an associated campaign.

Advertisements and conversions often appear on different sites. Sites cannot perform attribution without using some form of cross-site tracking. Having the browser involved makes it possible to perform attribution, privately. That is, the sites can perform attribution, but they cannot use what they learn to track people across sites.

### Overview

Our interface is as simple as possible, coming in two parts: impression registration and conversion.

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR4XmP4//8/AwAI/AL+GwXmLwAAAABJRU5ErkJggg==)

At impression time, information about an advertisement is saved by the browser in a write-only store. This includes an identifier for the ad and whether this was an ad view or an ad click.

At conversion time, information for aggregation is created based on the impressions that were previously stored. The converting site can select impressions based on the ad identifier and whether it was a click or a view.

- If there is a matching impression, then a value of one (1) is added to a histogram at the bucket that was specified for the ad at the time of the impression.
- If there was no matching impression, a value of zero (0) is added, making it impossible to distinguish from the outside whether an impression occurred.

The full histogram is submitted for aggregation using our [DAP](https://datatracker.ietf.org/doc/draft-ietf-ppm-dap/) service, which is a Multiparty Compute (MPC) system based on Prio for aggregation. This will use pre-configured DAP tasks for each advertiser. These tasks will produce an aggregated report for the advertiser.

The result of aggregation is a histogram of a predetermined size. Each bucket in the histogram will correspond to a single advertisement or a set of advertisements. The value in histogram buckets will include an estimate of how many of those ads resulted in conversions.

A key property of this aggregation is that the aggregation process ensures that private information from individual users is never visible to anyone. The advertiser only receives a report that includes differentially private information. That means that the values in each histogram bucket will contain some amount of noise.

### Impression Registration

A site can register ad impressions, either when the ad is shown or when the ad is clicked, at their discretion. They do this by generating and dispatching a CustomEvent as follows:

```javascript
navigator.privateAttribution.saveImpression({
  type: "view", // either "view" or "click"
  index: 3, // the histogram index for counting this impression
  ad: "moz-ads-feb-eijb", // a unique identifier for the ad placement
  target: "advertiser.example", // the advertiser site where a conversion will occur
});
```

This follows the WebIDL interface below:

```webidl
enum PrivateAttributionImpressionType { "view", "click" };
  dictionary PrivateAttributionImpressionOptions {
  PrivateAttributionImpressionType type = "view";
  required unsigned long index;
  required DOMString ad;
  required UTF8String target;
};

[SecureContext, Exposed=Window]
partial interface PrivateAttribution {
  [Throws] undefined saveImpression(PrivateAttributionImpressionOptions options);
}
```

This event contains a JSON string that includes information about where this ad is counted in the aggregation process (“index”). It also includes fields that allow the advertiser to identify this ad impression (“type”, “ad”, and “target”) when registering a conversion. The site that created the impression is automatically added to this data for use at conversion time.

This information is stored by the browser for a limited amount of time (suggestion: 30 days). Information is stored in a database that is associated with the “target” site. If cookies are cleared for that site, then the impression database is also cleared. Note that the target site cannot query this database directly.

### Conversion Reporting

A site that observes a conversion invokes a similar API to query for impressions.

The API is invoked by generating a custom event as follows:

```javascript
navigator.privateAttribution.measureConversion({
  // the task id of the aggregation job (given to the advertiser by Mozilla)
  task: "1s53f_aer0FJeX3j1f_avRedF03nFGIn30djnw2359s",
  // the number of buckets in the histogram for this task
  "size": 20,

  // only consider impressions within the last N days
  lookbackDays: 30,
  // the type of impression to match against (if omitted, match either)
  impression: "view",
  // a list of possible ad identifiers that can be attributed
  ads: ["moz-ads-feb-eijb"],
  // a list of sites where impressions might have been registered
  sources: ["publisher.example"]
});
```

This follows the WebIDL below:

```webidl
dictionary PrivateAttributionConversionOptions {
  required DOMString task;
  required unsigned long histogramSize;
  unsigned long lookbackDays = Infinity;
  PrivateAttributionImpressionType impression;
  sequence<DOMString> ads = [];
  sequence<UTF8String> sources = [];
};

[SecureContext, Exposed=Window]
partial interface PrivateAttribution {
  [Throws] undefined measureConversion(PrivateAttributionConversionOptions options);
};
```

This event contains a JSON map that is serialized into a string that includes a “type” field set to “conversion”, plus information that allows the browser to select saved impressions. This includes:

1. A “task” identifier that references a pre-configured DAP aggregation task. This is mandatory and needs to match the task identifier from the impression.
2. A “histogramSize” that indicates how many histogram buckets should be generated. This determines the maximum “index” that an impression can have; impressions do not match unless their “index” is less than this size.
3. The “lookbackDays” field optionally limits the impressions that are matched to those within the specified number of days. This can be omitted to match all stored impressions.
4. The “impression” field lists the type of impressions that is matched. This can be omitted to select either impression type.
5. The “ads” field lists the ad identifiers that need to match to select an impression. This can be omitted to select any ad.
6. The “sources” field lists the sites where impressions were registered. This can be omitted to select impressions from any site.

Impressions are also matched based on the “target” value, which needs to match the current site. If more than one matching impression is found, then the most recent one is selected.

The browser then generates a histogram with a value of 1 at the index identified in the impression record. This is secret-shared, encrypted, and submitted to an aggregation system using DAP and the identified task identifier.

A full solution here would allow for richer impression selection logic, the addition of a conversion value, and the ability to distribute that value to multiple impressions. For now, we want to concentrate on the simplest possible case.

### Example

An advertiser (“advertiser.example”) sells hats, shoes, and gloves. They decide to run campaigns for hats and shoes on a news publisher (“news.example”) and a social media site (“social.example”). They decide that shoes are worth additional investment and have three different ads that they hope will appeal to different audiences.

With two sites and four ads (one for hats, three for shoes), the advertiser decides to count conversions for each separately. They arrange to have a DAP task set up with 8 different histogram indices, with the allocation as follows:

| Index | Site | Product | Ad  |
| --- | --- | --- | --- |
| 0   | news.example | hats | 1   |
| --- | --- | --- | --- |
| 1   | news.example | shoes | 1   |
| --- | --- | --- | --- |
| 2   | news.example | shoes | 2   |
| --- | --- | --- | --- |
| 3   | news.example | shoes | 3   |
| --- | --- | --- | --- |
| 4   | social.example | hats | 1   |
| --- | --- | --- | --- |
| 5   | social.example | shoes | 1   |
| --- | --- | --- | --- |
| 6   | social.example | shoes | 2   |
| --- | --- | --- | --- |
| 7   | social.example | shoes | 3   |
| --- | --- | --- | --- |

We now follow the journey of a single user. This user is browsing the social network site and is shown shoe ad number 2. The site asks the browser to save an impression.

```javascript
navigator.privateAttribution.saveImpression({ type: "view", ad: "shoes", index: 2 });
```

This user does not interact with this ad, but they do click shoe ad 3 on the news site when it next appears.

```javascript
navigator.privateAttribution.saveImpression({ type: "click", ad: "shoes", index: 7 });
```

Meanwhile, back on the social network site, they are also shown an ad for a hat.

```javascript
navigator.privateAttribution.saveImpression({ ad: "hats", index: 4 });
```

They then add a pair of shoes to a cart, which the advertiser considers a conversion event worth measuring. The advertiser requests that a conversion report is generated.

```javascript
navigator.privateAttribution.measureConversion({
  task: "1s53f_aer0FJeX3j1f_avRedF03nFGIn30djnw2359s", size: 8,
  ads: ["shoes"], sources: ["news.example", "social.example"],
});
```

The browser then considers its store of impressions for “advertiser.example”, which looks something like this:

| Time | Type | Ad  | Source | Index |
| --- | --- | --- | --- | --- |
| 123456 | view | shoes | social.example | 2   |
| --- | --- | --- | --- | --- |
| 123458 | click | shoes | news.example | 7   |
| --- | --- | --- | --- | --- |
| 123459 | view | hats | social.example | 4   |
| --- | --- | --- | --- | --- |

The privacy budget is consulted for “advertising.example”, but as this is their first conversion report, that record does not exist.

The hat impression is most recent, but this does not match. The click impression for shoes is selected, resulting in an index of 7. The browser constructs a histogram with 8 buckets and values of \[0, 0, 0, 0, 0, 0, 0, 1\]. The browser updates the privacy budget for “advertising.example”, recording that one attributed conversion has been used.

The histogram values are then split into secret shares, encrypted, and submitted using DAP to the task that is configured on the aggregation service. The aggregation service validates the report and adds the report to those that it will later aggregate.

At some later time, when enough conversions have been reported, the aggregation service adds all the histograms that it has received, adds a fixed amount of noise to each bucket, and generates an aggregated report.

### Aggregation

We use the Prio system in a histogram sum mode, with reports submitted using the DAP protocol. The result is that each report is added to produce an aggregate.

In this case, each report consists of mostly zero values, with a single value that might be set to 1. We require a number of these reports to be aggregated in a batch (suggested minimum size: 50).

DAP validates the correctness of each report it receives before aggregating, ensuring that the aggregate is not spoiled by an invalid submission. Any false but otherwise valid submissions will still be aggregated; a full solution will include better affordances for managing things like fraudulent conversions.

Our DAP deployment is jointly run by Mozilla and [ISRG](https://divviup.org/blog/introducing-prio-services/). Privacy is lost if the two organizations collude to reveal individual values. We safeguard against this in several ways: trust in both organizations, joint agreements, and operational practices.

A full solution will require that advertisers — or their delegated measurement provider — receive reports from browsers, select a service, submit a batch of reports, and pay for the aggregation results, choosing from a list of approved operators.

For the trial, the results for each task will be sent to Mozilla’s telemetry systems, which will be used to access aggregated statistics.

### Differential Privacy

This design relies on differential privacy to provide a privacy guarantee that holds even if sites are malicious and attempt to abuse the API.

In practical terms, using differential privacy just means that some amount of noise will be added to the aggregated results.

The design uses a privacy budget that refreshes periodically (suggestion: weekly) for each advertiser site. Over each period, a small number of conversions will be reported (suggestion: 2). Once the limit is reached, conversions will be reported, but they will always include a value of 0 (which corresponds to an unattributed conversion).

Sites could track their use of the API and avoid asking for conversions once the limit is reached. However, this risks losing real conversions as an unattributed conversion will not count toward the limit. This uses the individual differential privacy framework as described in [this paper](https://arxiv.org/pdf/2405.16719).

The browser is responsible for tracking how many attributed conversions have been aggregated for each advertiser in each period. The number of attributed conversions for a site is a secret that does not reset if the site clears cookies. Therefore, this is stored separately from impressions.

The DAP system applies noise proportional to the maximum contribution that a single browser could add in each period (using suggested values, this is a maximum of 2 per week) and inversely proportional to an epsilon parameter (suggestion: 1), using an appropriate method. Suggested method is to use L1 sensitivity and the Laplace mechanism. For the trial, we do not place any requirements on how noise is applied, only that it is added to the final result that advertisers receive.

### Opt Out

This feature will be enabled by default with an option to disable it.

Having this enabled for more people ensures that there are more people contributing to aggregates, which in turn improves utility. Having this on by default both demands stronger privacy protections — primarily smaller epsilon values and more noise — but it also enables those stronger protections, because there are more people participating. In effect, people are hiding in a larger crowd.

An opt-in approach might enable weaker privacy protections, but would not necessarily provide better data in exchange. Having more data means both better measurement accuracy and an ability to add more noise on a per-person basis, meaning better privacy.
