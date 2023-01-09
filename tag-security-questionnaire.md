01.  What information might this feature expose to Web sites or other parties,
     and for what purposes is that exposure necessary?

The `requestStorageAccessForOrigin` API enables the removal of cross-site cookies. As such, the API provides a way for developers to re-gain access to cross-site cookies, albeit under further constraints.

When the top-level document calls `document.requestStorageAccessForOrigin(origin)` and is returned a resolving Promise, there are two effects:

* The top-level document begins to send `SameSite=None` cookies on subresource requests to `origin` when the request mode is CORS and credentials mode is `include`.
  * Note that a standard CORS response header is still required for the response to be readable.
* A nested Document on `origin` is able to gain access to its `SameSite=None` cookies when it calls `document.requestStorageAccess()` to indicate explicit opt-in. The prior grant of `requestStorageAccessForOrigin` waives the user activation requirement.
  * After this point, the same security rules as [the Storage Access API](https://github.com/privacycg/storage-access/blob/c4598484df6e77dc70a3d98499573ed44389f15e/tag-security-questionnaire.md) apply.

With these cookies, calls to `origin` can return user-specific information.

While this functionality comes with a risk of abuse by third parties for tracking purposes, it is an explicit goal of the API and a key to its design to not undermine the gains of cross-site cookie deprecation.

02.  Do features in your specification expose the minimum amount of information
     necessary to enable their intended uses?

Where possible, yes:

* Permission grants for storage access are double-keyed on a (top-level site, requested origin) basis, meaning that requesting documents will not be able to access cross-site cookies outside of the top-level site that they were granted access under.
* This feature allows access to all its `SameSite=None` cookies (if given permission), which may or may not exceed the minimum necessary amount of information that document needs. However, given the flexible nature of cookies it is very hard to determine which cookies a document needs and developer flexibility is an explicit goal of the API. Also, from a privacy perspective, passing any single high-entropy identifier such as a cookie across the site boundary is equivalent to full cross-site cookie access.

For security reasons, this API applies a few restrictions to how much information is exposed to a site that is granted storage access:

* CORS requirements mean that subresource requests will signal the caller to the embeddee (via the `Origin` header) and standard CORS opt-in requirements apply (i.e., a response header like `Access-Control-Allow-Credentials` must be returned for the response to be usable).
* For frame-level access to cross-site cookies, the embedded document must still invoke `document.requestStorageAccess()`, and its security requirements apply.
* This specification calls out that implementers should still follow SameSite rules when attaching cross-site cookies with storage access. This API does not intend to waive existing security protections.

03.  How do the features in your specification deal with personal information,
     personally-identifiable information (PII), or information derived from
     them?

As mentioned, the API enables sharing of information through cross-site cookies, but does not expand on that or deal directly with PII in any way.

04.  How do the features in your specification deal with sensitive information?

See above.

05.  Do the features in your specification introduce new state for an origin
     that persists across browsing sessions?
     
Yes, a new "top-level-storage-access" permission that is managed via the permissions API and is double-keyed on (top-level site, requested origin). Note that the double-key is expressed via the permission descriptor.

This should make it impossible for sites to access the new state across different top-level contexts.
     
06.  Do the features in your specification expose information about the
     underlying platform to origins?

No

07.  Does this specification allow an origin to send data to the underlying
     platform?
    
No
     
08.  Do features in this specification enable access to device sensors?

No

09.  Do features in this specification enable new script execution/loading
     mechanisms?
 
No
     
10.  Do features in this specification allow an origin to access other devices?

No

11.  Do features in this specification allow an origin some measure of control over
     a user agent's native UI?
     
While showing UI for top-level storage access prompts is left largely implementation-defined, this API can generally be expected to enable origins to spawn permission prompts detailing the top-level site and the requested origin in the UI.

Much like the equivalent concern with the Storage Access API, we have added a number of anti-abuse, spam and annoyance protections as outlined in the security considerations of the spec.

12.  What temporary identifiers do the features in this specification create or
     expose to the web?
     
None

13.  How does this specification distinguish between behavior in first-party and
     third-party contexts?

This specification is meant to be used in first-party contexts only, but it involves requesting access on behalf of a third-party (in other words, it expresses a cross-site relationship).

While the JS API itself will be accessible in a cross-site context, it will reject automatically when invoked outside the top-level document.

14.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?

The specification currently makes no explicit recommendation, as preferences may differ between user agents. It can generally be expected that in most user agents cross-site cookies are disabled in Private Browsing contexts, which would make it a natural fit for the API. However, exposing prompts to users in private browsing that request sharing of data between two sites may be viewed as intrusive. The API includes sufficient mechanisms for user agents to always deny storage access requests in private browsing.

15.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?

Yes

16.  Do features in your specification enable origins to downgrade default
     security protections?

To some extent, yes, albeit with strong controls to prevent accidental loss of protection. Deprecation of cross-site cookies prevents certain attacks as [outlined in detail in a recent SAA security analysis document](https://docs.google.com/document/d/1AsrETl-7XvnZNbG81Zy9BcZfKbqACQYBSrjM3VsIpjY/edit#heading=h.vb3ujl8dnk4q).

As compensating controls, the API:
* Requires frame-level access to be applied via explicit invocation of `requestStorageAccess`, such that controls in [the regular Storage Access API](https://github.com/privacycg/storage-access/blob/c4598484df6e77dc70a3d98499573ed44389f15e/tag-security-questionnaire.md) still apply.
* Requires any subresource requests be CORS-protected in order for cross-site cookies to be attached.
* Requires any such subresource requests to have been initiated by the top-level document, preventing unrelated frames from being granted access.

17.  How does your feature handle non-"fully active" documents?

It will reject calls to the API, as detailed in the spec.

18.  What should this questionnaire have asked?