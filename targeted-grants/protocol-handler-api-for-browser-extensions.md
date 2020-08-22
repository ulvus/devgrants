
# Targeted Grant: `Native Protocol Handler API for Browser Extensions`

**Issuer:** @lidel and @autonome

- [Project Description](#project-description)
- [Value](#value)
- [Deliverables](#deliverables)
- [Recommended Team](#recommended-team)
- [Detailed Requirements & Constraints](#detailed-requirements-constraints)
    - [Prior art: Web API for registering redirect-based handlers](#prior-art-web-api-for-registering-redirect-based-handlers)
    - [Prior art: browser extension registering a redirect-based handler](#prior-art-browser-extension-registering-a-redirect-based-handler)
    - [Prior art: browser extension registering a native handler](#prior-art-browser-extension-registering-a-native-handler)
    - [Prior art: Opera's native `ipfs://` URI backed by a Subdomain Gateway](#prior-art-operas-native-ipfs-uri-backed-by-a-subdomain-gateway)
    - [Next: native protocol handler API for browser extensions (this grant)](#next-native-protocol-handler-api-for-browser-extensions-this-grant)
- [Milestones & Funding](#milestones-funding)
- [Acceptance Criteria](#acceptance-criteria)
- [Resources](#resources)
- [Support and Funding](#support-and-funding)

## Project Description

Browser extensions are unable to register handlers for URIs such as `ipfs://<cid>`  

This grant aims to create a new API that enables browser extension to register
a _native_ protocol handler capable of returning streaming responses without redirecting to a third party server.


## Value

**Decentralization:** a generic Protocol Handler API could be used by IPFS and other protocols
such as [Dat](https://en.wikipedia.org/wiki/Dat_(software)) and [Secure Scuttlebutt](https://en.wikipedia.org/wiki/Secure_Scuttlebutt).

**Local and Offline:** browser extension developers could create more powerful extensions
running locally without the need for third party localhost apps or web services
for out of band processing and workarounds. Bringing back some of possibilities
that went away when Mozilla deprecated XUL-based extension ecosystem, 
but doing so within a well-defined, Origin-based security sandbox.

**IPFS:** native protocol handler API would enable [IPFS Companion](https://github.com/ipfs/ipfs-companion) browser extension
to return data fetched from IPFS network without the need for running a local HTTP Gateway.
It would provide same addressing and Origin isolation with embedded js-ipfs
as with go-ipfs running on localhost. Origin would not be tied to HTTP transport,
but based purely on the root [CID](https://github.com/ipld/cid),
enabling seamless transition between IPFS backends
while leveraging integrity guarantees of content addressing.


## Deliverables

- Vendor-agnostic API specification
- Chromium/Blink implementation

## Recommended Team

[Igalia](https://www.igalia.com/) is an open source consultancy specialized
in the development of innovative projects and solutions, including the web platform. 

They are experienced contributors to major browser engines familiar with both codebases
and standardization processes. One of most prominent success stories is [CSS Grid](https://www.igalia.com/2017/04/04/Shipping-CSS-Grid-Layout-in-major-browsers.html),
which they championed and implemented in Blink and WebKit.

On the IPFS side, grant will be supported by @lidel and @autonome.

## Detailed Requirements & Constraints

This grant tries to build on top of what is already possible,
pushing web platform and browser extensions forward. 

Below we describe the current state of the platform and prior art
around protocol handlers in browser extensions. 

Finally, we describe what is missing and in separate section list
acceptance criteria for this grant.

### Prior art: Web API for registering redirect-based handlers

A generic Web API for registering custom protocol handler exists
([`navigator.registerProtocolHandler`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/registerProtocolHandler)),
but it is heavily limited.

```js
navigator.registerProtocolHandler('web+ipfs',
                                  'https://example.com/?uri=%s',
                                  'IPFS handler')
```

Above example:

- Works only when executed on matching Origin
    - User needs to be on `example.com` for registration to work
    - Displays user prompt, asking for confirmation if a handler should be registered
- Requires unknown protocols names to be prefixed with `web+` ([registerProtocolHandler#Permitted_schemes](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/registerProtocolHandler#Permitted_schemes))
   - Firefox whitelisted `ipfs://` and `ipns://` ([bug 1428446](https://bugzilla.mozilla.org/show_bug.cgi?id=1428446))
   - Chromium requires prefix `web+ipfs://` ([bug 651311](https://bugs.chromium.org/p/chromium/issues/detail?id=651311),
     [intent-to-implement](https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/29sFh4tTdcs/K4XroilVBAAJ) is blocked by [whatwg/html#3998](https://github.com/whatwg/html/issues/3998))
- In the end, is just a gloried HTTP redirect
  - opening `web+ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi` will redirect to a web-based handler at `http://example.com?uri=web%2Bipfs%3A%2F%2Fbafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi`

### Prior art: browser extension registering a redirect-based handler

#### Firefox: `manifest.json/protocol_handlers`

Since Firefox 59 it is possible for a browser extension to perform  `navigator.registerProtocolHandler`-like handler registration during own install by defining handled URI schemes upfront in [`manifest.json/protocol_handlers`](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/protocol_handlers).

This automatically registers a redirect-based protocol handler when browser extension is installed and supports DWeb schemes, such as `ipfs://` and `ipns://` without `web+` prefix (whitelisted in [bug 1428446](https://bugzilla.mozilla.org/show_bug.cgi?id=1428446)).

It removes the need for user to visit specific website, 
but remains a thin wrapper on top of `navigator.registerProtocolHandler` API, 
with the rest of its limitations: it is impossible for a browser extension
to return a byte array representing response for requested URL.

This API is not supported by Chromium ([bug 883274](https://bugs.chromium.org/p/chromium/issues/detail?id=883274)).

### Prior art: browser extension registering a native handler

#### Muon-based Brave

Before Brave switched to Chromium in 2018, we got basic experimental protocol handler working in Muon. It did not support streaming but was an important milestone for IPFS project. Details can be found in [ipfs-companion/issues/312](https://github.com/ipfs-shipyard/ipfs-companion/issues/312).

#### Firefox Nightly + libdweb

IPFS, Dat and others collaborated with Mozilla on prototyping native protocol handler
under project called [libdweb](https://github.com/mozilla/libdweb). 
Old libdweb experiments produced [PoC Protocol API](https://github.com/mozilla/libdweb/blob/master/Readme.md#protocol-api)
and we got [PoC extension using the  API](https://github.com/ipfs-shipyard/ipfs-companion/pull/533) to work,
unfortunately those APIs remained experimental and did not land in regular Firefox Nightly. 
Over time, Gecko codebase moved forward breaking the demo in latest Nightlies.

There was an effort to reimplement an async iterator version of the PoC Protocol API
in the upstream codebase ([bug 1271553](https://bugzilla.mozilla.org/show_bug.cgi?id=1271553)),
but it does not seem to be a priority for Mozilla at this time.

### Prior art: Opera's native `ipfs://` URI backed by a Subdomain Gateway

[Opera for Android 57] shipped native support for `ipfs://` and `ipns://` URIs.
It introduced a custom protocol handler that translates requests made to `ipfs://` to a URL at a public [subdomain gateway].
This way each content root loaded from IPFS gets a unique Origin and is sandboxed from other ones.

While the implementation done by Opera did not use browser extensions, the
similar approach (native `ipfs://` backed by HTTP gateway) could be leveraged
in other browsers if we empower extensions with capability to register custom
protocol handlers.

[Opera for Android 57]: https://blog.ipfs.io/2020-03-30-ipfs-in-opera-for-android/
[subdomain gateway]: https://docs.ipfs.io/how-to/address-ipfs-on-web/#subdomain-gateway

### Next: native protocol handler API for browser extensions (this grant)

The work in scope of this grant is to leverage lessons from the past experiments
and create a general purpose Protocol Handler API that can be used by any browser extension.
Majority of browser vendors support a variant of Chromium's [Manifest V2](https://developer.chrome.com/extensions/manifest)
browser extensions (Firefox calls them [WebExtensions](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions)).
Chromium-based vendors are vocal about being interested in improving IPFS support. 
To maximize the value created by this grant Chromium codebase should be used as implementation target.

If possible, the API should be compatible with `manifest.json/protocol_handlers` API already present in Firefox 
(registering handler on extension install) but with option to omit `uriTemplate` and provide a self-hosted,
programmatic handler via to-be-created `chrome.*.registerProtocol` API instead. 

See _Milestones_ and _Acceptance Criteria_ for more details.

## Milestones & Funding

**Estimated Funding Amount:** 36000 USD (based on hours agreed between Igalia and Protocol Labs)

Note: due to the nature of this exploratory grant, in the event of a Milestone
landing being blocked completely before the completion of the full hours, the
billing may be under the estimated total and grant recipient will bill only
hours worked.

**Milestones:**

| Milestone No. | Milestone Description                                                        |
| ---           | ---                                                                          |
| 1             | Enhance `registerProtocolHandler()` to whitelist ”ipfs” and ”ipns” protocols |
| 2             | Support `protocol_handlers` key in Chromium extensions                       |
| 3             | Implement a native protocol handler API for browser extensions               |

## Acceptance Criteria

<!-- What are the acceptance criteria for each milestone and for the final deliverables? These should be as objective as possible. They will be used to determine whether or not a grantee will receive payment for work completed for a milestone.  -->

### Milestone 1: Enhance `registerProtocolHandler()` to whitelist ”ipfs” and ”ipns” protocols

- Restart discussion and find a browser vendor consensus at
  https://github.com/whatwg/html/issues/3998
- Update https://html.spec.whatwg.org to reflect these changes.
- Add/Update web platform tests at https://github.com/web-platform-tests/wpt
  accordingly.
- Restart discussion at
  https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/29sFh4tTdcs/K4XroilVBAAJ
- Ensure the intent is up-to-date with the latest criteria at
  https://www.chromium.org/blink/launching-features
- Submit new patches against the latest upstream and land them into Chromium.
- Any extra tasks suggested by reviewers to get this accepted (e.g.  meeting
  with API owners, adding a runtime flag, whitelist more protocols, switch to a
  blacklist, remove the title argument, security improvements, etc).

### Milestone 2: Support `protocol_handlers` key in Chromium extensions

- Coordinate with Microsoft and Google on this proposal, in particular:
  https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/280
  https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/230
- Ensure that at least the ”ipfs” and ”ipns” protocols are allowed.
- Make any change to Mozilla’s implementation if necessary
- Analyze code and get in touch with Chromium’s community to understand the
  process to extend https://developer.chrome.com/extensions, propose a new API,
  how they are used in Chromium-based browsers other than Chrome, if a runtime
  flag is needed, etc.
- Write PoC patches adding support for this in Chrome’s extensions.
- Submit patches upstream and land them, with any extra tasks suggested by
  reviewers to get this accepted (e.g. installation prompt for security, tests,
  etc).

### Milestone 3: Implement a native protocol handler API for browser extensions

- API specification is approved by IPFS project
    - Origin is based on the content root
    - supports streaming responses by means of async iterators
    - reuses HTTP semantics for caching, content type, headers and error codes
    - likely similar to libdweb’s proposal and an extension of
      `protocol_handlers` from task Milestone 2.
- API implementation in form of patches for Chromium codebase
    - experimental build to test integration with IPFS Companion browser extension
    - allows JS running in browser extension context to register `ipfs://` and `ipns://` protocol handlers, process every request made with them and return arbitrary bytes
- API proliferation
    - released under [PL's Permissive License Stack](https://protocol.ai/blog/announcing-the-permissive-license-stack/) or a license suggested by the Chromium project
    - discussed with Microsoft, Mozilla and Chromium projects
    - patches can be integrated into the codebase of other Chromium-based browsers such as Edge or Brave.
    - patches submitted to the upstream Chromium / Blink projects

## Resources

Additional resources that might be helpful for an implementer who is working on this project.

- Whitelisting Dweb protocols
    - [/arewedistributedyet/issues/23](https://github.com/arewedistributedyet/arewedistributedyet/issues/23)
    - [/whatwg/html/issues/3935](https://github.com/whatwg/html/issues/3935)
- Firefox
  - old libdweb experiments
      - [PoC protocol API](https://github.com/mozilla/libdweb/blob/master/Readme.md#protocol-api) / [discussion](https://github.com/mozilla/libdweb/issues/2)
      - [PoC extension using above protocol handler API](https://github.com/ipfs-shipyard/ipfs-companion/pull/533)
   - new libdweb experiments
     - [Bug 1271553 (libdweb-protocol): Add ability to implement programmable custom protocol handler](https://bugzilla.mozilla.org/show_bug.cgi?id=1271553)
- Chromium
  - [Issue 651311: registerProtocolHandler: support version control system and decentralized protocols](https://bugs.chromium.org/p/chromium/issues/detail?id=651311)
  - [Issue 883274: Extensions API should implement manifest.json/protocol_handlers](https://bugs.chromium.org/p/chromium/issues/detail?id=883274)
  - [Issue 64100: Extensions should be able to register for protocols](https://bugs.chromium.org/p/chromium/issues/detail?id=64100)
- IPFS Companion browser extension
    - [Support Custom Protocols in WebExtension](https://github.com/ipfs-shipyard/ipfs-companion/issues/164)
    - [libdweb experiment: protocol handler API](https://github.com/ipfs-shipyard/ipfs-companion/pull/533)
    - [Embedded JS IPFS in Brave](https://github.com/ipfs-shipyard/ipfs-companion/issues/716)

## Support and Funding

This grant is sponsored by Protocol Labs.
