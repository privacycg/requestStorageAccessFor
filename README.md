# Extending the Storage Access API Explainer

## Authors:
* [Matt Reichhoff](https://github.com/mreichhoff) (mreichhoff@chromium.org)
* [Kaustubha Govind](https://github.com/krgovind) (kaustubhag@chromium.org)
* [Johann Hofmann](https://github.com/johannhof) (johannhof@chromium.org)

## Participate

* [Issues on this repo are welcome!](https://github.com/mreichhoff/requestStorageAccessForSite/issues)
* Feedback on [the issue in the storage access repo](https://github.com/privacycg/storage-access/issues/107) is also welcome!

## Introduction

Enabled-by-default cross-site cookie access is in the process of being deprecated by several major browsers. Multiple substitutes have been proposed, like [the Storage Access API](https://webkit.org/blog/8124/introducing-storage-access-api/) and the [SameParty cookie attribute](https://github.com/WICG/first-party-sets#sameparty-cookies-and-first-party-sets) in the [First-Party Sets](https://github.com/WICG/first-party-sets) proposal.

With the proposal to [abandon active development of the `SameParty`](https://github.com/WICG/first-party-sets/issues/92) cookie attribute in favor of requiring a call to the Storage Access API to enable cross-site cookie use cases, this document briefly discusses how the existing Storage Access API works and how First-Party Set membership can be applied as part of it.

While the integration of Storage Access API with First-Party Sets alone will solve some use cases, many legacy issues with 3rd party cookie usage involve instances where user interaction within an `<iframe>` is difficult to retrofit, e.g. because of the use of images or script tags requiring cookies.

Both Firefox and Safari have run into these issues before and solved them through the application of an internal-only "requestStorageAccessForOrigin" API([1](https://bugzilla.mozilla.org/show_bug.cgi?id=1724376),[2](https://github.com/WebKit/WebKit/commit/e0690e2f6c7e51bd73b66e038b5d4d86a6f30909#diff-1d194b67d50610776c206cb5faa8f056cf1063dd9743c5a43cab834d43e5434cR253)), which is applied on a case-by-case basis by custom browser scripts (Safari: [3](https://github.com/WebKit/WebKit/blob/a39a03d621e441f3b7ca3a814d1bc0e2b8dd72be/Source/WebCore/page/Quirks.cpp#L1065),[4](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Quirks.cpp#L1217) Firefox: [5](https://phabricator.services.mozilla.com/D129185),[6](https://phabricator.services.mozilla.com/D124493),[7](https://phabricator.services.mozilla.com/D131643)). While such an internal API should not be treated as setting precedent, it could be evidence that such an API would be useful for developers.

Accordingly, this document proposes a variant of this API as an extension to the Storage Access API to improve its ergonomics, with a requirement of increased trust between browser and sites as outlined in the Security & Privacy Considerations section.


## Goals
* Enable use cases for which `SameParty` cookies were proposed, but in a manner that better aligns with existing browser APIs.
* Within the bounds of guardrails like First-Party Sets, enable greater flexibility and easier adoption of the Storage Access API.
* Allow user agents to define storage access policies that are informed by site author-specified information.

## Non-goals
* Maintaining parity with the existing `SameParty` cookie proposal, particularly in characteristics like synchronicity, is not a goal.
* Enforcement of a specific browser treatment or behavior for StorageAccessAPI requests (either the existing `requestStorageAccess` or the proposed `requestStorageAccessForSite`) as part of the implementation defined steps is not a goal. Every user agent should still be free to take custom steps, including prompting users or other heuristics.

## Current requestStorageAccess Behavior
The existing Storage Access API [specifies](https://privacycg.github.io/storage-access/#the-document-object) `requestStorageAccess`, but [delegates](https://privacycg.github.io/storage-access/#ua-policy) the decision on whether to grant access to the browser. Firefox and Safari have each implemented their own set of requirements, such as whether the user has previously interacted with the requester in a top-level context; the number of existing grants for the origin in the session; whether the user consents to the sharing; and others.

While a heuristics-based approach is possible, browsers could also choose to use First-Party Sets membership information to inform their behavior. For example, browsers with FPS integration may choose to automatically grant or deny requests based on First-Party Set membership. In a hypothetical scenario, if `fps-member2.example`, embedded in an `<iframe>` on `fps-member1.example`, calls requestStorageAccess, and the two domains are in the same First-Party Set, the call could resolve successfully and access could be granted.

The same call made by `other-party.example`, also embedded in an iframe on `fps-member1.example`, where `other-party.example` is not in the same First-Party Set, could be rejected and cookie use denied. 

For those browsers that prompt the user, FPS membership could also potentially inform such behavior (for example, as an anti-abuse measure).

This is compatible with the requestStorageAccess specification. The user activation and other requirements would remain in place; this behavior is simply [the browser logic step (step 9 of the spec)](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess#conditions_for_granting_storage_access), and other browsers would continue with prompting or other measures that match the product expectations of their user base.


```
<!--top-level site: fps-member1.example; fps-member2.example is in the same First-Party Set-->
<html>
<body>
 <iframe src="https://fps-member2.example">
  <script>
   document.addEventListener('click', function() {
     document.requestStorageAccess().then(/*auto-granted; same-party*/);
   });
  </script>
 </iframe>
 <iframe src="https://other-party.example">
  <script>
   document.addEventListener('click', function() {
     document.requestStorageAccess().catch(/*auto-rejected; not same-party*/);
   });
  </script>
 </iframe>
</body>
</html>
```



## Proposed Extension: requestStorageAccessForSite

Since the `requestStorageAccess` API was [originally designed](https://webkit.org/blog/8124/introducing-storage-access-api/) for authenticated embeds, it has requirements that are perhaps uniquely well-suited for that category of use-cases. Specifically, it is only possible for the embedded party to request access, and only from within `<iframe>` elements that have received user interaction. However, these restrictions place adoption costs on websites that have functionality deployed across multiple sites, where cross-site subresources may include images or JavaScript files instead of `<iframe>`-embedded documents. [A similar discussion](https://github.com/privacycg/storage-access/issues/3) previously resulted in the existing `requestStorageAccess` API operating at the page level, rather than frame-only.

This document proposes a similar API, `document.requestStorageAccessForSite`, which would allow the embedding site to request access it knows it needs on behalf of its embedded sites. 

This new API would be very similar to the existing `requestStorageAccess`. It would still require activation, though of the top-level document; would still delegate to per-browser logic, so that each browser can customize the experience to their user' expectations; and would functionally be equivalent to if `requestStorageAccess` had been called by the passed-in domain with the same top-level context.

This API could be treated as similar in principle to browser-specific compatibility measures, implemented in [Safari](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Quirks.cpp#L1131-L1163) and [Firefox](https://searchfox.org/mozilla-central/rev/287583a4a605eee8cd2d41381ffaea7a93d7b987/dom/base/Document.cpp#17051), where an internal API is invoked, based on site-based allowlists, that requests cross-site cookie access on behalf of embedded sites.

Like with the proposed `requestStorageAccess` implementation described in the previous section of this document, granting access could be determined by First-Party Set membership.

Requiring user interaction in `<iframe>` elements helps the original `requestStorageAccess` API deter spam and abuse from first-parties and their embedded third-party scripts. To compensate for lack of this protection, browsers should require additional trust signals to grant storage access via the proposed API, such as FPS membership. This is discussed in Privacy & Security Considerations.


## Key scenarios
### Embedded Sites
This is the standard use of the `requestStorageAccess` API; usage would be equivalent to that implemented in other browsers. The browser-specific logic, as described above, would simply be a check on First-Party Set membership.

See the numerous [examples available elsewhere](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess#examples) for sample code; this proposal does not modify developer-facing use of the existing `requestStorageAccess` API.

### Top-level Requests on Behalf of Another Party
In contrast with the existing `requestStorageAccess` API, the proposed extension would allow non-iframe use, and would afford more control to the top-level party over what access is requested when.

With grant logic based only on First-Party Set membership, example use could be:

```
<!--
Top-level site: fps-member1.example. fps-member2.example is in the same First-Party Set. other-party.example is not.

Note that all <script> tags below would be new post-3rd party cookie deprecation.

Assume fps-member2.example has previously set two cookies:
Set-Cookie: sameSiteLax=123; SameSite=Lax 
Set-Cookie: sameSiteNone=456; SameSite=None; Secure
-->
<html>
<head>
  <script>
    document.requestStorageAccessForSite('https://fps-member2.example')
     .then(
           /*not called;no activation.*/)
     .catch(/*called due to top-level site lacking activation at load time*/);
  </script>
</head>

<body>
<button id='play-button'></button>
<script>
  const playButton = document.getElementById('play-button');
  playButton.addEventListener('click', function(){
    document.requestStorageAccessForSite('https://fps-member2.example')
     .then(
       /*
       called;has activation, same First-Party Set is sufficient.
       Cookie `sameSiteNone=456` available. Cookie `sameSiteLax=123` is not.
       Image tags or other assets could be requested: 
       */
       let img = document.createElement('img'); 
       img.src='https://fps-member2.example/profile_pic.png';
       document.body.appendChild(img);
     )
     .catch(/*not called due to lacking activation*/);
  
    document.requestStorageAccessForSite('https://other-party.example')
     .then(/*for v1, rejected; not in the same First-Party Set*/)
     .catch(/*called due to not being in the same First-Party Set*/);
  });
</script>
<!--initial page load: would not have cookies; see event handler above-->
<img src='https://fps-member2.example/profile_pic.png'>
<!--
Would have access to its cross-site cookies, after the button is clicked and the process runs as expected. Future page-loads would remember this for some TBD period of time.
-->
<iframe src='https://fps-member2.example'></iframe>
<!--
Would not require a separate call to requestStorageAccessForSite, because of site, not origin, scoping.
-->
<iframe src='https://sub-domain.fps-member2.example'></iframe>
</body>
</html>
```


## Detailed design discussion
### Proposed Draft Spec Addition

The proposed spec could include a set of steps for the browser to follow, much like is done with <code>[requestStorageAccess](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess#conditions_for_granting_storage_access)</code>. The spec could include a function that takes a <code>string</code> as the site:


```
function requestStorageAccessForSite(site)
```

Where a draft set of steps could be:
1. If the browser is not processing a user gesture, reject.
1. If the document already has been granted access, resolve.
1. If the document has a null origin, or if the requested domain is invalid or has a null origin, reject.
1. If the document's frame is not the main frame, reject.
1. If the requested origin is equal to the main frame's, resolve.
1. Check any additional rules that the browser has. Reject if some rule is not fulfilled.
    1. If the browser implements First-Party Sets, this could entail a check equivalent to `IsSameParty(top_level_site, requested_site)`
    1. Other browsers could continue with their own rules, e.g., requiring prior first party interaction with the requested origin, prompting the user, etc. 
1. Grant future subresource requests access to cookies and store that fact for the purposes of future calls to requestStorageAccessForSite()

### Plural vs Singular API

One could imagine either a singular API:

```
// follows the existing 1x1 pattern established by requestStorageAccess
// returns a Promise similarly
requestStorageAccessForSite("site.example")
```

Or a plural one:

```
// allows, for example, use of Promise.all()
requestStorageAccessForSites(["site1.example","site2.example"])
```

Given the increased complexity with potential user prompts, and the 1x1 nature of the existing `requestStorageAccess` API, the singular version is recommended. Note that it is also simpler to switch from singular to plural than from plural to singular, should that ever become necessary. 


### Site vs Origin Scope

The existing `requestStorageAccess` API is scoped to site for the top-level page in both Safari and Firefox, but to origin for the embedded requester in Firefox, but site in Safari.

This has been [the subject of debate](https://github.com/privacycg/storage-access/issues/39). This proposal is to scope the grant to site for both sides of the call, thus avoiding a requirement of repeated calls for origins like `www.site.example` and `site.example`.


## Considered alternatives

### Browser-Specific Allowlists

As discussed in the introduction, both Firefox and Safari have implemented an internal-only “requestStorageAccessForOrigin” API([1](https://bugzilla.mozilla.org/show_bug.cgi?id=1724376),[2](https://github.com/WebKit/WebKit/commit/e0690e2f6c7e51bd73b66e038b5d4d86a6f30909#diff-1d194b67d50610776c206cb5faa8f056cf1063dd9743c5a43cab834d43e5434cR253)), that is applied on a case-by-case basis by custom browser scripts (Safari: [3](https://github.com/WebKit/WebKit/blob/a39a03d621e441f3b7ca3a814d1bc0e2b8dd72be/Source/WebCore/page/Quirks.cpp#L1065),[4](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Quirks.cpp#L1217) Firefox: [5](https://phabricator.services.mozilla.com/D129185),[6](https://phabricator.services.mozilla.com/D124493),[7](https://phabricator.services.mozilla.com/D131643)).

This approach is not preferred, as it favors websites that have access to the corresponding browser's developers, and may not produce equitable outcomes. In addition, it does not allow site authors to proactively fix issues without interacting with browser developers.

### Forward Declaration

Forward declaration of storage access requirements [remains under discussion](https://github.com/privacycg/storage-access/issues/83). This proposal is not intended to replace (or otherwise take a stance on) that option, which may still be relevant in the future. Instead, this proposal attempts to resolve adoption considerations that aren’t directly resolved by the forward declaration-based design, which requires a top-level navigation to the embedded origin and is intended to address identity/login use-cases.


### HTTP Header Access Requests

Using the Storage Access API introduces a dependency on JavaScript for a site wanting to use cookies within a First-Party Set context. While this may be a fairly small selection of sites, the sites and their clients may not have previously required JavaScript, which increases the effort for adoption.

In a similar way to using the `allow` attribute on an `iframe` to enable specific features for a domain map to an equivalent Permissions Policy, it would be possible to provide an equivalent for a storage access call.

For example, the JavaScript call:

```
document.requestStorageAccessForSite('https://fps-member2.example')
```

Could be equivalent to an HTTP header, possibly using [permissions policy syntax](https://developer.chrome.com/en/docs/privacy-sandbox/permissions-policy/):

```
Permissions-Policy: storage-access=(self "https://fps-member2.example")
```

While this option may be attractive in the future, and would be doable in a First-Party Set membership-driven approval system, it is outside the scope of this document, which builds on the existing JavaScript API. Such an option is instead considered a potential future work item.


## Privacy and Security Considerations

By exposing a new access-granting API, especially one that relaxes the current `<iframe>` requirement of requestStorageAccess and allows for arbitrary domains to be passed in, care must be taken not to open additional security issues or abuse vectors. It is easy to imagine an untrusted top-level domain requesting access on behalf of an unrelated site. Such access could enable CSRF, clickjacking, or other attacks.


### Elevated Trust Requirement

To prevent unrelated sites from requesting access, browsers should seek additional trust signals to enable the API to resolve successfully. First-Party Sets is one such mechanism; it could ensure that a relationship exists between the caller and the passed-in site. However, sites designated as part of [the "service" subset](https://github.com/krgovind/first-party-sets#defining-a-set-through-use-case-based-subsets) should not be allowed to gain access on behalf of other sites in the set. Browsers that don’t support First-Party Sets could utilize other mechanisms, like user prompts, allowlists, denylists, or other heuristics. 


### Cross-Site Protections

Existing cross-site protections, like the `SameSite` cookie attribute, will continue to be respected; access granted by `requestStorageAccessForSite` would apply only to `SameSite=None` cookies. By ensuring a default `SameSite` setting of at least `Lax`, browsers can ensure that the embedded resources opted into cross-domain sharing by setting `SameSite` to `None`. 

However, this protection (alongside `x-frame-options` and others) may not be sufficient, since sites may globally set `SameSite=None` cookies that are required only on a subset of resources that are intended to be consumed across site boundaries. Additional explicit opt-in, perhaps by ensuring that cookies are not sent except on CORS-enabled endpoints, or by ensuring that only certain domains in a First-Party Set are authorized to successfully call `requestStorageAccessForSite`, may be desirable.


#### CSRF Considerations

A side effect of disabling `SameSite=None` cookies is that attacks like CSRF become significantly harder to carry out. While the existing `requestStorageAccess` API already allows a mechanism to opt out of this protection (especially due to the fact that the API [unlocks](https://github.com/privacycg/storage-access/pull/27) cross-site cookies for all requests to subresources on the requesting site, not just the specific document making the request), `requestStorageAccessForSite` could be used more broadly due to its relaxation of the `<iframe>` requirement. Additionally, `requestStorageAccessForSite` is invoked by the embedder, as opposed to the embedded origin which gets access to cross-site cookies. This may make additional opt-in requirements for embedded resources, like those described above, more attractive.


### Abuse Prevention

There is also a risk of abuse of the API by top-level documents, for example by attempting to get user consent to data sharing with an unrelated third-party, or by using prior interaction requirements to infer browsing history. This is especially true because of the ubiquity of third-party scripts included in top-level contexts. Gating access on First-Party Sets (which also guarantee mutual exclusivity, preventing a single domain from being present in multiple sets) is one mechanism by which this concern can be mitigated.  Additional anti-abuse mechanisms, especially for those user agents that do not support First-Party Sets, could include:



*   Consuming (rather than merely requiring) user activation on the top-level website, which would prevent repeated attempts at gaining access without additional user activity.
*   Limiting the number of calls to the API on a given page load.
*   Monitoring usage in order to quiet permission prompts or auto-reject requests by disruptive sources.
*   Injecting timing noise into resolution of promises to prevent side channel attacks (for example: an adversary using the time to resolution to infer that the user had been prompted, and that the other browser requirements, like prior interaction with the domain at the top level, had been fulfilled).

## Stakeholder Feedback / Opposition

*   TBD

## References & acknowledgements

Many thanks for valuable feedback and advice from:

* [Artur Janc](https://github.com/arturjanc)
* The existing [Storage Access API spec](https://privacycg.github.io/storage-access/), the [MDN Storage Access API documentation](https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API) and the [Safari documentation](https://webkit.org/blog/8124/introducing-storage-access-api/) were all instrumental in authoring this document.
