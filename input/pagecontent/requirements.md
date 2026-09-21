### Requirements

This document describes the requirements for all servers that are part of the [HL7 terminology ecosystem](ecosystem.html).
Note that systems do not need to conform to these requirements to be described as 'FHIR Terminology servers',
but they do need to conform to these requirements to be part of the ecosystem.

#### How to read this page

Every requirement on this page is enforced by one or more of the [test cases](testcases.html), and the
test cases are the definitive statement of what the requirement means: where this page and a test case
disagree, that is a defect in one of them, and should be reported. Each section names the test suites
that cover it, so that an implementer can go from a rule to the tests that check it.

Most requirements apply to every server. Some apply only to a server that supports a particular code
system or a particular feature, and those are stated conditionally - "if a server supports X, then it
SHALL ...". In the test cases these conditional requirements are gated by a **mode**: a test gated on a
mode you do not pass to the runner does not run, and does not count as either a pass or a failure. So
a server declares which of these sections apply to it by which modes it asks to be tested in. See
[Modes](testcases.html#modes).

The test cases are written in R5 but run against R4 servers as well, with the runner converting
requests and responses on the fly. Where a requirement takes a different form in R4, it says so, and
[R4 and the Test Cases](r4.html) has the detail.

### General Requirements

#### FHIR Version

A server speaks one version of FHIR, and the test runner discovers which:

* The server SHALL either support the `$versions` operation at the root, or return a
  `CapabilityStatement` from `/metadata` that populates `CapabilityStatement.fhirVersion`. A server
  that does neither cannot be tested, and cannot be registered
* The server SHALL support either R4 or R5. Other versions are not supported by the ecosystem
* The server SHALL be consistent: the version it reports is the version in which it will accept
  requests and return responses

#### HTTP and Errors

*(covered throughout; the `http-code` property on a test case says what is expected)*

* The server SHALL support the JSON format (`application/fhir+json`). XML is not required
* The server SHALL accept operation parameters as a `Parameters` resource POSTed to the operation
  endpoint. GET forms are used in places by clients but are not required by the ecosystem
* The server SHALL return HTTP 200 when the operation completed - including when the answer is a
  negative one. A `$validate-code` that finds the code invalid is a successful operation, and returns
  200 with `result = false`
* The server SHALL return a 4xx status with an `OperationOutcome` when it could not carry out the
  operation at all: the value set could not be resolved, the request was malformed, the code system
  is unknown to a `$subsumes` call, the expansion is too costly. The test cases state the expected
  status as `4xx` rather than a specific code, so any 4xx is accepted
* The server SHALL NOT return a 5xx status for any of the test cases. A 5xx means the server broke

#### Issues and Messages

Every error, warning and comment the server produces is carried as an `OperationOutcome.issue`, either
in a returned `OperationOutcome` or in the `issues` parameter of an operation response. For all of them:

* Each issue SHALL have `severity`, `code`, `details.coding`, and `details.text`
* `details.coding` SHALL include a coding from `http://hl7.org/fhir/tools/CodeSystem/tx-issue-type`.
  This is what lets a validator process the issue correctly, and it matters more than `issue.code`,
  for which more than one value is often accepted
* Issues that relate to a particular part of the request SHALL carry `expression` naming it, so that
  a validator can locate the issue in the resource being validated
* `diagnostics` MAY be populated; the test cases ignore it
* The exact wording of a message is not fixed. The test cases match some messages by fragments, and a
  server may supply its own wording for the rest through an externals file - see
  [Test Output](testcases.html#test-output). What is fixed is the severity, the issue type, and the
  facts the message has to name (typically the code and the code system)
* Servers MAY populate the `http://hl7.org/fhir/StructureDefinition/operationoutcome-message-id`
  extension with the server's own identifier for the message. Only tx.fhir.org is required to

#### Defending the Server

A server in the ecosystem is exposed to requests it did not write, from clients it does not control, so
it has to be able to say no. *(suites: `big`, `regex-bad`)*

* Where an expansion would be larger than the server is prepared to produce, the server SHALL return a
  4xx with an issue whose `tx-issue-type` is `too-costly`. What the limit is, and how it is calculated,
  is up to the server; that there is a limit, and that exceeding it produces this answer, is not
* Where a value set's imports are circular, the server SHALL detect it and return a 4xx with an issue
  whose `tx-issue-type` is `vs-invalid`, rather than looping
* Where a regex filter is capable of catastrophic backtracking, the server SHALL either evaluate it
  safely and return the right answer, or refuse the request with an error. It SHALL NOT hang. Both
  answers pass the tests

### Metadata

*(suite: `metadata`)*

#### $versions

* The server SHOULD support `$versions` at the root, returning the FHIR versions it supports. It is the
  first thing the test runner asks for, and it is the only way for a server that speaks more than one
  version of FHIR to say so
* If the server supports `$versions`, it SHALL declare it in `CapabilityStatement.rest.operation`

#### CapabilityStatement

* The server SHALL return a CapabilityStatement from `{root}/metadata`
* It SHALL populate `url`, `version`, `name`, `title`, `status` (`active`), `date`, and `kind`
  (`instance`)
* It SHALL populate `software.name`, `software.version` and `software.releaseDate`. The `name` is what
  the test runner uses to identify the server in its output
* It SHALL populate `fhirVersion`, and `rest[mode = server].security.service`
* It SHALL include `application/fhir+json` in `format`
* It SHALL include a `CapabilityStatement.instantiates` value of
  `http://hl7.org/fhir/CapabilityStatement/terminology-server`
* It SHALL declare, under `rest.resource` for `CodeSystem`, the operations `lookup`, `subsumes` and
  `validate-code`
* It SHALL declare, under `rest.resource` for `ValueSet`, the `read` and `search-type` interactions and
  the operations `expand` and `validate-code`
* Where the server supports the [caching protocol](#caching), it SHALL declare the `cache-control`
  operation at the **system** level in `rest.operation`. This declaration is the only way a client
  discovers that the protocol is available
* It SHALL carry a
  [feature extension](http://hl7.org/fhir/uv/application-feature/StructureDefinition/feature) for
  `CodeSystemAsParameter`, saying whether the server accepts CodeSystem resources in the `tx-resource`
  parameter. The feature SHALL be present; its value may be `true` or `false`
* It SHALL carry a feature extension reporting the version of the test cases the server passes. A
  server may update this after release if it comes to pass a later version, but is not required to

#### TerminologyCapabilities

* The server SHALL return a TerminologyCapabilities from `{root}/metadata?mode=terminology`
* It SHALL populate `version`, `name`, `title`, `status` and `date`
* It SHALL list every code system it supports in `TerminologyCapabilities.codeSystem.uri`, with the
  versions in `codeSystem.version.code`. Code systems SHALL be listed here whether or not they are
  available through `/CodeSystem` search - this list, not the `/CodeSystem` endpoint, is what the
  ecosystem uses to decide whether to send a request to a server
* It SHALL list, in `expansion.parameter`, at least the parameters `activeOnly`,
  `check-system-version`, `count`, `displayLanguage`, `excludeNested`, `force-system-version`,
  `includeDefinition`, `includeDesignations`, `offset`, `property`, `system-version` and `tx-resource`
* It SHOULD also list any of the optional parameters it supports there, in particular
  `default-valueset-version`. Note that caching is **not** declared here: `cache-id` is no longer an
  expansion parameter, and support for caching is declared by the `$cache-control` operation in the
  CapabilityStatement - see [Caching](#caching)

### Content

#### Supporting CodeSystems

A '<code>supported</code>' CodeSystem is any code system that the server supports correctly for calls to `$expand`, `$validate-code`, and `$lookup`.
A '<code>pre-defined</code>' CodeSystem is any value set that the server makes available through the `/CodeSystem` endpoint.

There are two kinds of servers that are made available to the ecosystem: general purpose terminology servers that support 
arbitrary CodeSystem resources, and servers that are code system specific - they only support one code system (or a select
short list of CodeSystems). 

Servers are encouraged to make all the code systems that they support available on the `/CodeSystem` endpoint (search/read) but 
they do not have to, and code systems such as SNOMED CT and LOINC often are not. But they must be listed in the 
TerminologyCapabilities statement so that the ecosystem knows which code systems the server supports.

* The TerminologyCapabilities SHALL list all the predefined code systems that the server supports in `TerminologyCapabilities.codeSystem.uri`, and all the versions in `TerminologyCapabilities.codeSystem.version.code`. Code systems SHALL be listed here whether or not they are available through code system search 
* The server SHOULD make all the code systems it supports available through the `/CodeSystem` endpoint for search and read operations
* The server SHALL support the ```url``` and ```version``` search parameters

Note that it's the bigger code systems such as SNOMED CT and LOINC that might not be available through `/CodeSystem`. Also note that the 
terminology ecosystem does not make use of any search parameters

* A server does not *have to* make all the code systems it supports available at `/CodeSystem`
* CodeSystems available at `/CodeSystem` MAY have `content = not-present`; the tools will not consider this when choosing whether to try using the code system (uses `TerminologyCapabilities.codeSystem.uri`)

Servers are encouraged to ensure that all code systems conform to the ShareableCodeSystem Profile found in the [CRMI specification](https://build.fhir.org/ig/HL7/crmi-ig/), but this is not a technical requirement for being part of the ecosystem.

* Servers SHOULD generally allow multiple resources for the same canonical URL with different Resource.version, but this is subject to business rules on the server 
* Servers do not need to support update/create, and the ecosystem never makes use of these interactions.

It's up to the server how to manage what content they support and implement. Servers can choose to support update/create if they want, though it SHOULD only do so for authenticated clients. 

* Servers SHALL ensure that they only send correct results for the code systems for which it is registered as 'authoritative' (e.g., not allow not appropriately authorised users to change the results by posting resources)

* All predefined code systems SHALL have a web representation that is appropriate for a human to look at (see below)

#### Passing CodeSystem resources in requests 

Terminology servers can choose to accept CodeSystems in the [tx-resource parameter](https://jira.hl7.org/browse/FHIR-33944). 
General purpose servers SHALL do this (and are required to do this to pass the general tests).

* Servers SHALL indicate their support or not for passing CodeSystem resources in the tx-resource parameter using the
  `CodeSystemAsParameter` feature in the CapabilityStatement (see above)

Terminology servers SHOULD support passing CodeSystem supplements, particularly language packs. Servers that don't support 
language packs should only choose not to support language packs when there is governance over the use of language translations,
and only by negotiation with the terminology ecosystem managers (see also notes below)

* Servers SHALL indicate their support or not for passing CodeSystem supplements in the tx-resource parameter using [tbd]

* Servers intended to be used as the primary server for the validator and IG publisher SHALL accept CodeSystems in the 
  [tx-resource parameter](https://jira.hl7.org/browse/FHIR-33944). (e.g. this rule does not apply servers registered with the ecosystem)

#### Code system Functionality

Servers are required to support code system supplements. Specifically, this means:

* Servers SHALL not ignore supplements, though they MAY choose to return errors rather than process them correctly

Where a server makes [language specific authoritative claims](ecosystem.html#language-specific-claims) in the ecosystem registration (the `languages` property):

* The server SHALL host the content of the claimed code systems (`complete` or `fragment`), not just the supplements - it will receive the whole operation, not just the language-specific part
* The server SHALL support each claimed language for the code systems claimed under it: `$expand`, `$validate-code`, and `$lookup` SHALL return correct displays and designations in that language (typically via loaded supplements). This is verified by the language routing test cases, not by the coordination server

Servers are required to support the following properties in the CodeSystem resource:

* `CodeSystem.caseSensitive` SHALL be supported: validation SHALL correctly check case. In the case of non-case sensitive code systems, expansions SHOULD just contain the code as defined (in the code system or the value set enumeration), and not all the case variants that could be generated. *(suite: `case`)*

* `CodeSystem.valueSet`. Tbd what this means

* `CodeSystem.hierarchyMeaning`. If the code system defines a hierarchy, both `$expand` and `$validate` SHALL correctly handle the hierarchy when interpreting filters and validating codes 

* `CodeSystem.compositional`. A server SHALL not accept compositional grammars for codes unless the CodeSystem is marked as Compositional. Servers that do not know the grammar for a CodeSystem that is marked as compositional SHALL note this in validation errors for the CodeSystem

* `CodeSystem.versionNeeded`. If a CodeSystem says that version is needed, the `$validate-code` operation SHALL check that version is populated, and return an error if it's not

* `CodeSystem.content`. Servers SHALL not process `$expand` or `$validate-code` requests on CodeSystems that have `content = not-present` or `example`. Servers SHALL reflect `content = fragment` in an error message if the code is not valid against a fragment. Specifically, where the code system is a fragment, a code that is *not* found SHALL NOT be reported as invalid: the answer is that the server cannot tell, and the message SHALL say that the code system is a fragment *(suite: `fragment`)*

* `CodeSystem.supplements`. Servers SHALL not mistake supplements and code systems for each other.

#### Concept Properties

A number of concept properties defined at
[concept-properties](http://hl7.org/fhir/concept-properties) change how a concept behaves, and a server
SHALL honour them. The rule for recognising them is the same in each case, and is tested in detail for
`notSelectable` *(suite: `notSelectable`)*:

* Where the CodeSystem declares the property with the canonical `uri` from `concept-properties`, the
  server SHALL recognise it **whatever local `code` the code system gave it**
* Where the CodeSystem declares no property definition at all, the server SHALL recognise the property
  by its standard code
* Where the CodeSystem declares the property with a *different* `uri`, it is a different property, and
  the server SHALL NOT treat it as the standard one

The properties that matter to the ecosystem are `status`, `inactive`, `deprecated`, `notSelectable`,
`definition`, `parent` and `child`. See
[Inactive, Deprecated and Not-Selectable Codes](#inactive-deprecated-and-not-selectable-codes) for what
the first four mean.

#### Supporting Value Sets 

A '<code>supported</code>' value set is any value set that can be used in `$expand` or `$validate-code` operations, including value sets imported into other value sets, and including implicit value sets. A '<code>pre-defined</code>' value set is any value set that the server makes available through the `/ValueSet` endpoint.

All servers are required to fully support value sets as defined in this document (per below). 

* The server SHALL make all the predefined value sets it supports available through the `/ValueSet` endpoints for search and read operations
* The server SHALL support the ```url``` and ```version``` search parameters
* The server SHALL support the `_summary` search parameter

Servers are encouraged to ensure that all value sets conform to the ShareableValueSet Profile found in the [CRMI specification](https://build.fhir.org/ig/HL7/crmi-ig/), but this is not a technical requirement for being part of the ecosystem.

* Servers SHOULD generally allow multiple resources for the same canonical URL with different Resource.version, but this is subject to business rules on the server 
* Servers do not need to support update/create, and the ecosystem never makes use of these interactions.

It's up to the server how to manage what content they support and implement. Servers can choose to support update/create if they want, though it SHOULD only do so for authenticated clients. 

* All predefined value sets SHALL have a web representation that is appropriate for a human to look at (see below)

##### Passing ValueSet resources in requests 

* Servers SHALL accept ValueSets passed in the [tx-resource parameter](https://jira.hl7.org/browse/FHIR-33944) to both `$expand` and `$validate-code` operations.
* Servers SHALL use these value sets when resolving imports in other ValueSet resources
* Servers SHALL indicate their support for passing ValueSet resources in the tx-resource parameter in `TerminologyCapabilities.expansion.parameter`

##### ValueSet functionality 

* Servers SHALL support `ValueSet.compose` 
* Servers SHALL support `ValueSet.compose.include` and `ValueSet.compose.exclude` 
* Servers SHALL support imported value sets (for both include and exclude)
* Servers SHALL support extensionally defined value sets (by enumerating codes in ValueSet.compose.in|exclude.concept)
* Servers SHALL support intensionally defined value sets (using filters)
* Servers SHALL support all the filters defined in the base specification for all code systems
* Servers SHALL return an error if a ValueSet uses a filter they do not understand 
* Servers SHALL only return an existing expansion if it is the correct expansion for the definition of the value set

The ecosystem makes no rules - at this time - about the handling of value sets that have an expansion with no definition.

#### Supporting ConceptMaps

*(suites: `translate`, `translate2`)*

A server that supports `$translate` holds ConceptMaps in the same way it holds CodeSystems and
ValueSets, and the same rules apply:

* The server SHALL accept ConceptMap resources in the `tx-resource` parameter, and use them for
  `$translate` requests in the same call
* The server SHOULD make pre-defined ConceptMaps available through the `/ConceptMap` endpoint for read
  and search, and SHALL support the `url` and `version` search parameters if it does
* The server SHALL support ConceptMaps that declare `sourceScope` / `targetScope` and ConceptMaps that
  do not. A ConceptMap with no scope is still usable; it is only the server's ability to *find* it
  without being told which map to use that is affected

#### Human Representation 

* All pre-defined code systems and value sets SHALL have a web representation (as above) 

The content on that page MAY be static or active; it is at the discretion of the server to decide what's on the page, but it SHOULD be more than just the json/xml for the resource (and it isn't limited to information in the resource, e.g. the server MAY choose to make additional process/provenance/context information available)

The server MAY choose to make this content available at the end-point for the relevant resource. e.g. a request for `{root}/CodeSystem/123` with an ```Accept``` header of 'application/fhir+json' returns the resource, and the same URL with an ```Accept``` header of 'text/html' returns a web page suitable for human consumption. Servers are not required to do this; they MAY choose to make the content available elsewhere.

If the server chooses to make them available elsewhere, it SHALL populate the extension ```http://hl7.org/fhir/StructureDefinition/web-source``` in any resources it makes available with a `valueUrl` where the web view can be found. This SHALL be populated when the CodeSystem and ValueSet are read, and also in any `$expand` of the value set (just for the root value set in this case).

### Common Parameters

These apply to `$expand` and `$validate-code` alike.

* The server SHALL support the [tx-resource](https://jira.hl7.org/browse/FHIR-33944) parameter for passing related terminology, per the rules above
* If a server receives a terminology resource it does not process correctly, it SHALL return an error

* The server SHALL support supplements for the purposes of designations in different languages
  * Clarification: Servers SHALL not ignore supplements; if they don't support a relevant supplement (per the rules above), they SHALL return an error that they cannot process the supplement (e.g. if it is passed in a tx-resource parameter)
  * Whether servers actually accept and use supplements for the purposes of designations is a matter for negotiation with the server's relevant user base and whether there are other arrangements in place for supporting translation (e.g. SNOMED CT, LOINC)

* The server SHALL support `system-version`, `check-system-version` and `force-system-version`, and
  SHALL observe the difference between them: `system-version` supplies a default that anything explicit
  in the value set overrides, `force-system-version` overrides the value set, and `check-system-version`
  produces an error if the value set asks for a different version. See
  [Code System Versions](#code-system-versions)

### Caching

A terminology client - the validator, the IG publisher - validates and expands thousands of times
against the same value sets and code systems, and many of them are ones the server does not already
have, because the IG being built is what defines them. Re-sending those definitions on every call is
the single largest source of wasted traffic in the ecosystem. The `$cache-control` protocol lets a
client send each resource to the server once and refer to it by url thereafter.

The protocol is defined in the FHIR Tools IG - see
[Terminology Caching](http://hl7.org/fhir/tools/terminology-caching.html) for the protocol and the
reasoning behind it, and the
[$cache-control OperationDefinition](http://hl7.org/fhir/tools/OperationDefinition-cache-control.html)
for the formal definition. That IG is normative for the protocol; what follows is what the ecosystem
requires of a server that is part of it.

Note that this replaces the earlier `cache-id` *parameter*. The cache-id is now issued by the server
and carried in an HTTP header, and the parameter is withdrawn: servers SHOULD NOT accept it, and it is
no longer listed in `TerminologyCapabilities.expansion.parameter`.

#### Supporting the protocol

* Servers SHOULD support `$cache-control`. This is not yet a SHALL, but it makes a very large
  difference to build times and to network use, and it is expected to become one
* A server that supports it SHALL declare the `cache-control` operation at the **system** level in
  `CapabilityStatement.rest.operation`. A client uses the protocol only against a server that
  advertises it, and inlines the resources on every request against any other - correct, just not
  optimised
* The operation declares `affectsState = true`; servers SHALL accept `start` and `end` by `POST`

Note that declaring the operation has an immediate consequence: **the test runner uses it**. A server
whose CapabilityStatement advertises system-level `cache-control` will have each test suite's `setup`
resources front-loaded with `mode=start` and then referred to by url under the returned cache-id,
instead of being re-sent with every request. The runner does not pass `sealed`, so it takes the
server's default, which SHOULD be `true` - a sealed cache holding exactly the suite's setup. A server
that advertises the operation but does not implement it correctly will fail the whole suite rather than
one test, so declare it when it works, not before.

#### Identifying the cache

* The cache-id SHALL be allocated by the **server**, and returned in the `cache-id` output parameter of
  a `mode=start` call. A client does not invent one. This is what makes it possible for the server to
  say authoritatively whether a cache-id it is handed is one it actually has
* On subsequent requests the client sends the cache-id as the **`X-Cache-Id` HTTP header**, not as an
  operation parameter, so that it is transport metadata: readable by proxies and load balancers, and
  by the server before the body is parsed
* Caches are scoped to the endpoint and FHIR version they were started on. A server SHALL NOT honour a
  cache-id on an endpoint other than the one that issued it

#### Sealed and unsealed caches

* A `mode=start` response SHALL report `sealed`, so that a client never has to assume a default
* A **sealed** cache - the default - holds exactly the resources front-loaded in the `start` call.
  Resources sent inline on later requests are used for that request and SHALL NOT be added to the cache
* An **unsealed** cache (`sealed = false`) SHALL accumulate each resource the server sees under that
  cache-id - whether sent as `tx-resource` or as the primary `valueSet` / `codeSystem` - the first time
  it is seen, and resolve it by reference thereafter
* A client that asks for an unsealed cache takes on the consequences of shared mutable state on the
  server, in particular the need not to issue overlapping requests that populate and read the same
  cache concurrently. Sealing avoids that class of problem, which is why it is the default

#### Reporting a cache the server does not have

The three cases - never issued, released, timed out - are indistinguishable to the client unless the
server distinguishes them, so:

* Where a `$validate-code`, `$expand` or any other operation carries an `X-Cache-Id` the server does
  not have, the server SHALL fail the request with HTTP **404** and an `OperationOutcome` whose issue
  carries `cache-id-unknown` from `http://hl7.org/fhir/tools/CodeSystem/tx-issue-type`. It is the
  **coded issue**, not the status, that a client keys on: `cache-id-unknown` means the client's cache
  is gone and it should start a new one, where an unknown value set is an authoring error. A server
  that reports a lost cache as a content error sends implementers hunting for a mistake that is not
  there
* `mode=end` for a cache the server does not have SHALL NOT be an error. It returns HTTP 200, because
  the client's intent - that the cache be gone - is already satisfied
* Servers SHOULD record why each cache-id they retired went away, and say which in the diagnostics:
  never issued by this server, released by the client, or timed out through disuse. Only the last is
  fixed by checking more often, and the three otherwise look identical on the wire. A bounded record of
  the most recently retired cache-ids is enough; an id old enough to have fallen off the end honestly
  reports as never issued

#### Keeping a cache alive

A client can depend on a server-side cache it has not used for a long time, because its own local cache
is answering everything; the failure then shows up as the first rare code that does reach the server,
long into a build. `mode=check` exists for this.

* `mode=check` SHALL return HTTP **200** whatever the answer - it is not an error to ask about a cache
  that is gone. A client polling its cache has to be able to tell "the server is up and says my cache
  is gone" from "I could not reach the server": the first means start a new cache, the second means
  retry, and collapsing them into a failed request loses exactly that
* For a valid cache, the response SHALL carry `valid = true` and `sealed`, and SHOULD carry
  `resource-count` and `idle` (the idle time as it was *before* the check reset it)
* The response SHOULD carry `timeout`, the server's idle timeout in seconds, where the server is
  willing to state one. A client has no other way to discover it, and without it can only guess at how
  often to check
* For a cache the server does not have, the response SHALL carry `valid = false` and an `outcome`
  parameter holding an `OperationOutcome` with the same `cache-id-unknown` issue that a request using
  the cache-id would have failed with
* **A successful check counts as use**, and SHALL reset the cache's idle timer. A client that asks
  whether its cache is still there is by definition a client that still wants it, so no separate
  keep-alive mechanism is needed

#### Populating the cache

* A client may send the same resource more than once under the same `url` + `version` - front-loaded at
  `start`, and again on later requests - and the server SHALL tolerate it. A client is not expected to
  track what it has already registered
* Every copy sent under a given `url` + `version` is required to be identical, so a server MAY treat
  any copy as authoritative - first, last, or any other. A server MAY verify that repeated copies
  match and reject a request that redefines a `url` + `version` with different content, but a client
  SHALL NOT rely on that check being made
* The cache holds resource **definitions**, not expansions. A value set sent with an inline `expansion`
  has that expansion cached as supplied
* Where the server supports [batches](#batch-validation) against an unsealed cache, every resource
  supplied anywhere in the batch - as `tx-resource` or as an entry's primary `valueSet` / `codeSystem` -
  SHALL be populated into the cache **before any entry is evaluated**. An entry may therefore refer by
  url to a resource supplied by a different entry, whatever order the two appear in, and population
  SHALL NOT depend on whether individual entries succeeded. The batch's effect on the cache is
  all-or-nothing at the level of the batch, not the entry. None of this applies to a sealed cache,
  which does not grow

#### Caching and versionless references

Caching does not change which version a versionless reference resolves to, but it makes a disagreement
about it both more durable and harder to see.

* A resolution made once against the cache is reused for the life of the cache, so a single
  disagreement between client and server about what "latest" means is frozen in and repeated on every
  request that uses that cache-id, rather than being re-evaluated each time
* Because the client refers to a cached resource by url alone after the first send, such a mismatch is
  invisible on the wire
* Content in the ecosystem SHOULD therefore pin versions explicitly on references between terminology
  resources wherever the exact version matters. Caching makes an existing versionless-resolution
  disagreement worse, not better

### $expand

*(suites: `simple-cases`, `parameters`, `properties`, `exclude`, `search`, `notSelectable`, `inactive`,
`deprecated`, `fragment`, `big`, `overload`, `other`, `errors`, `default-valueset-version`, `version`,
`regex-bad`, `language`)*

#### Core behaviour

* The server SHALL expand a value set given by `url`, by `valueSet` (a resource in the request), or by
  a resource passed in `tx-resource` and named by `url`
* The server SHALL echo all parameters - including ones it assumed a value for - in
  `ValueSet.expansion.parameter`
* The server SHALL report every code system version it actually used, as a `used-codesystem` expansion
  parameter with the value `{url}|{version}`. Where a supplement was used, it SHALL also report
  `used-supplement`; where another value set was imported, `used-valueset`. This is how a client knows
  what the expansion actually depended on
* The server SHALL populate `expansion.identifier` and `expansion.timestamp`
* The server SHOULD populate `expansion.total` where it knows the total
* The server SHOULD return hierarchical expansions when possible (this is not a technical requirement, but comes up as important to authors)
* The server SHALL support `excludeNested`, which forces a flat expansion
* An unknown code named in an enumerated `include` is skipped, not an error: the expansion contains
  the codes that do exist, and no issue is raised
* Where the expansion is knowingly incomplete - the underlying grammar is unbounded, or the server
  returned a base subset - the expansion SHALL be marked as unclosed rather than presented as complete

#### Expansion parameters

* The server SHALL support `count` and `offset`. `offset` is used by the ecosystem's clients only with
  the value 0, but both are tested
* The server SHALL support `activeOnly`, `includeDesignations`, `includeDefinition`, `property` and
  `designation`
* The server SHALL support `displayLanguage`, and the `Accept-Language` header, and
  `ValueSet.language`, as specified in [Languages](languages.html)
* The server SHALL support `filter` (text search). The ecosystem makes no requirements about the
  *quality* of the text search - only that the parameter is supported and that obvious matches are
  found and obvious non-matches are not *(suite: `search`)*
* The server SHALL support `abstract` in the sense described under
  [notSelectable](#inactive-deprecated-and-not-selectable-codes)
* The server SHOULD support `default-valueset-version`, which supplies a version for imported value
  sets that are referenced without one. Where it is supplied and names a value set version that does
  not exist, the server SHALL return an error rather than silently falling back
  *(suite: `default-valueset-version`)*

#### Filters

The server SHALL support all the filters defined in the base specification, for all the code systems it
supports, and SHALL observe the distinctions between them *(suites: `simple-cases`, `tho`)*:

* `is-a` includes the code named in the filter as well as everything below it
* `descendent-of` excludes the code named in the filter - that is the only difference from `is-a`
* `is-not-a` excludes the code named in the filter *and* everything below it; everything else in the
  code system, including the code above it and other branches, is included
* `child-of` includes the direct children only, and not the code named in the filter
* `=`, `in`, `not-in`, `exists` and `regex` on concept properties
* Hierarchy SHALL be followed however the code system states it - by nesting, or by a `parent` /
  `subsumedBy` property. A code system may state more than one parent for a concept, which nesting
  cannot express, and the filters SHALL honour all of them
* A filter the server does not understand, or a filter with no value where one is required, SHALL
  produce an error - not an empty or a partial expansion

#### Properties in expansions

*(suite: `properties`)*

* Where the expansion request asks for properties - with the `property` parameter, or with
  `ValueSet.compose.property` in the value set definition - the server SHALL return them
* `ValueSet.compose.property` SHALL be supported in both its forms: an enumerated list of property
  codes, and `*` meaning every property the code system defines
* The server SHALL declare the properties it returned in `ValueSet.expansion.property`, giving each
  one's `code` and, where it has one, its canonical `uri`
* The server SHALL return the values in `ValueSet.expansion.contains.property`
* **In R4**, `ValueSet.compose.property`, `ValueSet.expansion.property` and
  `ValueSet.expansion.contains.property` do not exist as elements. An R4 server SHALL read and write
  them as the cross-version extensions
  `http://hl7.org/fhir/5.0/StructureDefinition/extension-ValueSet.compose.property`,
  `http://hl7.org/fhir/5.0/StructureDefinition/extension-ValueSet.expansion.property` and
  `http://hl7.org/fhir/5.0/StructureDefinition/extension-ValueSet.expansion.contains.property`. This
  is a requirement, not an option: an R4 server that ignores the request or drops the response is
  not conformant. [R4 and the Test Cases](r4.html#properties-in-expansions) has worked examples of
  both

#### Imports, exclusions and multiple versions

* The server SHALL support `compose.exclude` in all its forms: excluding enumerated codes, excluding by
  filter, and excluding by importing another value set *(suite: `exclude`)*
* Excluding everything that was included is not an error - it produces an empty expansion
* The server SHALL support a value set that includes two versions of the same code system, and SHALL
  keep the versions distinct: a code enumerated from one version is not thereby selected from the
  other, and an exclusion scoped to one version does not remove the code from the other
  *(suite: `overload`)*
* Where an include names no version, the server resolves it to its default for that code system; where
  a `system-version` parameter is also supplied, it is only a default, and SHALL NOT override a version
  the value set states explicitly

### $validate-code (ValueSet)

*(suites: `validation`, `version`, `language2`, `permutations`, `overload`, `notSelectable`, `inactive`,
`deprecated`, `errors`, `fragment`, `other`, `tho`)*

* The server SHALL support validating `code` + `system` (+ `version`) (+ `display`), `Coding`, and
  `CodeableConcept`. A CodeableConcept SHALL be validated as a whole: the result is true if any of its
  codings validates, and the issues report what was wrong with the others
* The server SHALL support the [mode/valueSetMode](https://jira.hl7.org/browse/FHIR-41229) parameter
* The server SHALL support language correctly (same locations/rules as `$expand`)
* The server SHALL support the [inferSystem](https://jira.hl7.org/browse/FHIR-41431) parameter. Where
  `inferSystem` is true and the value set contains the same code in two code systems, the server SHALL
  report the ambiguity as an error rather than choosing one
* The server SHALL support `lenient-display-validation`, and SHALL distinguish the two answers: with it
  set, a display that is wrong but recognisable produces a warning; without it, an error
* The server SHALL support `abstract`, which says whether the caller will accept a code that is not
  selectable

Return parameters:

* the server SHALL return a result parameter, along with a message summarising the reason why if result = false
* any errors and warnings SHALL be itemised in an  `issues` parameter with paths to the actual locations so a validator can locate the issue correctly 
* any issue entries in the OperationOutcome SHALL have a severity, type, expression, details.coding, and details.text. The coding SHALL be taken from the `http://hl7.org/fhir/tools/CodeSystem/tx-issue-type` system, and helps validators process the errors correctly. The diagnostics property may be populated;
this is ignored by the test cases 
* the correct value for issue.type and issue.details.coding may be found in the text cases. At least with regard to issue.type, the correct code is sometimes unclear, and more than one type is accepted
* the server SHALL return the code system and code against which validation was based (`system` and `code` parmaeters)
* The server SHOULD return the version against which validation was based (there are corner case exceptions, e.g. where there is no version on the code system)
* the server SHOULD return a display for the code (`display` parameter) but this is not always possible (e.g. some codes do not have displays) or required for some causes of validation failure
* The server SHALL return a `x-caused-by-unknown-system` parameter for each code system it did not support. This helps validators inform users of missing resources
* The server SHOULD return a `normalized-code` parameter where appropriate (e.g. case insensitive code systems, code systems with complex grammars)
* The server SHOULD return an issue with tx issue type ```processing-note``` when it has not fully validated the code e.g. an SCT expression against the MRCM 

The difference between "this code is not valid" and "this code is not in this value set" is a real one,
and the `tx-issue-type` coding SHALL say which: `invalid-code` for the first, `not-in-vs` for the
second. A message SHALL name both the code and the value set or code system it was checked against.

### $validate-code (CodeSystem)

The same rules apply, less the value set. `CodeSystem/$validate-code` is tested separately throughout
(the test cases call it `cs-validate-code`), because a server can get one right and the other wrong.

* Where the server holds the code system but the code is not in it, the answer is `result = false` with
  an `invalid-code` issue, not an error
* Where the server does not hold the code system at all, that is an error

### Batch validation

*(suite: `batch`)*

* The server SHOULD support batch validation: a `$validate-code` request carrying repeated `validation`
  parameters, each one a `Parameters` resource describing one code to validate. This is what makes
  validating a large resource affordable
* Parameters given at the top level of the request - `url`, `tx-resource`, `lenient-display-validation`
  and so on - apply to every validation in the batch
* A parameter given inside an individual `validation` overrides the top-level one for that validation
  only
* The response SHALL contain one `validation` parameter for each one in the request, **in the same
  order**, each holding either the `Parameters` that `$validate-code` would have returned, or an
  `OperationOutcome` where that one validation could not be performed
* One bad validation in a batch SHALL NOT fail the batch: the others are still answered

### $lookup

*(suites: `simple-cases`, `parameters`, `omop`, `UCUM`, `icd-11`, `snomed`)*

`$lookup` is not used heavily by the ecosystem's tools, so there are few tests, but what it returns has
to be right.

* The server SHALL return `display`, and the `name` of the code system, and its `version`
* The server SHALL support the `property` parameter, including `property=*`, which asks for every
  property the code system defines for the concept
* Where `property=*` is asked for, the server SHALL return `definition`, the hierarchy (`parent` and
  `child` properties), and the standard concept properties it knows - `inactive`, `notSelectable`,
  `status` - as well as the code system's own properties
* The server SHALL return designations, with their `use` and, where known, their `language`
* The server SHOULD return `abstract`
* `code` SHALL be echoed as it was asked for, not normalised. Where the server normalises the code, the
  normal form belongs in the `code` *property*, which is what that property is for
* Where the code is not valid, the server SHALL return a 4xx and an `OperationOutcome`

### $subsumes

*(suites: `simple-cases`, `tho`, `snomed`, `UCUM`, `mimetypes`, `langcodes`, `tx.fhir.org`)*

* The server SHALL support `$subsumes` on `CodeSystem`, in both forms: `system` + `codeA` + `codeB`,
  and `codingA` + `codingB`
* The server SHALL return an `outcome` parameter with one of `equivalent`, `subsumes`,
  `subsumed-by` or `not-subsumed`
* `equivalent` SHALL be returned when the two codes are the same code, and when they are two spellings
  of the same thing in a code system whose grammar makes that possible
* Subsumption SHALL be transitive: a code subsumes its distant descendants, not just its direct
  children
* Subsumption SHALL follow the hierarchy however the code system states it. Where the code system
  states hierarchy with a `subsumedBy` or `parent` property rather than by nesting, the server SHALL
  read it, including where a concept has more than one parent
* Two codes with a common ancestor, but neither on the other's path to it, are `not-subsumed`. So are
  two parents of the same code
* Where either code is not valid, or not in the code system, or the version or system named is one the
  server does not have, the server SHALL return a 4xx with an `OperationOutcome`. The message SHALL
  name the code and the code system
* Where the code system's structure does not settle the question, the server SHALL say so rather than
  guessing. Finding no relationship is not the same as establishing that there is none: see
  [SNOMED CT](#snomed-ct) and [Mime Types](#mime-types-bcp-13), where this distinction is tested in
  detail

### $translate

*(suites: `translate`, `translate2`, `omop`, `tx.fhir.org`)*

A server that holds ConceptMaps SHALL support `ConceptMap/$translate`. This section is new, and the
tests are correspondingly more detailed than the text; where they disagree, the tests are right.

#### Naming the concept to translate

* The server SHALL accept the source concept as `sourceCode` + `sourceSystem`, as `sourceCoding`, or as
  `sourceCodeableConcept`
* **In R4** the same parameters are named `code` + `system`, `coding` and `codeableConcept`, and the
  target system is `targetsystem` (lower case s) rather than `targetSystem`. An R4 server SHALL accept
  the R4 names
* The server SHALL accept `targetSystem`, and SHOULD accept `targetScope` (`target` in R4), to
  constrain what the answer may be

#### Choosing the ConceptMap

* Where the request names a map with `url`, the server SHALL use that map and no other. Naming a map
  does not exempt it from the scope rule below: a client cannot put an out-of-scope code into a map's
  scope by nominating the map, so the answer is still that the map does not apply
* Where the request names no map, the server SHALL find the ConceptMaps whose `sourceScope` and
  `targetScope` fit the request, and use them. A server MAY decline to do this - not every server is
  willing to choose a map on the client's behalf - in which case it SHALL say so rather than returning
  an empty result
* A map applies to a source code only where that code is within the map's `sourceScope`.
  `sourceScope` *limits the scope of the map*, so a code outside it is not the map's business at all:
  the map is not a candidate - whether the server chose it or the client named it - and in particular
  its `unmapped` SHALL NOT fire for that code. Were it otherwise, a map with `unmapped mode = fixed`
  would answer for every code in the code system its group names, whatever its author declared, and
  `sourceScope` would have no effect on any map that has an `unmapped` - which is most of them
* A ConceptMap reached as another map's `unmapped` / `other-map` target is **consumed by that
  delegation**, and SHALL NOT also be consulted as a candidate map in its own right. The delegating map
  swallows it. Were it consulted independently as well, every `other-map` delegation would yield a
  duplicate match whenever the two maps share a scope - which is the normal case, since a map normally
  delegates to one covering the same ground - and `other-map` would be unusable
* Every match SHALL report in `originMap`, as `{url}|{version}`, the map the server **entered** - not
  the map that ultimately supplied the value. Where a delegation was followed, the match belongs to
  the delegating map. **In R4** this parameter is named `source`
* Where the server followed a map other than the one it started from, it SHALL report the map it ended
  up in as a `used-conceptmap` parameter. That, and not a second match, is how the server discloses
  where a delegated value came from

#### The result

* `result` SHALL be true where any mapping was found, and false where none was. There are three
  different negative answers, and a server SHALL distinguish them:

  | | `result` | `match` |
  | --- | --- | --- |
  | a map applies and says explicitly that there is no mapping | `true` | one, with `noMap = true` and no `concept` |
  | a map applies, has no element for the code, and its `unmapped` fires | `true` | the unmapped concept |
  | no map applies at all | `false` | none |

  The third is a successful operation with a negative answer, so it returns HTTP 200 with a `message`
  saying why - not a 4xx
* Each mapping SHALL be returned as a `match` parameter, whose parts are `concept` (the target Coding),
  `relationship`, `originMap`, and `sourceConcept` (the Coding that was translated). **In R4** the
  relationship is reported in `equivalence`, using the R4 codes (`equivalent`, `narrower`, `wider`,
  `relatedto` ...), and the R5 `relationship` element SHALL NOT be present; in R5 it is the other way
  round
* `sourceComment` SHALL be returned where the ConceptMap element carries a comment
* The server SHALL support all the relationship types, and map them correctly between R4 and R5

#### No-map and unmapped

* Where the ConceptMap explicitly states that a code has no mapping - `noMap` in R5, a target with no
  code in R4 - the server SHALL return a `match` with `noMap = true` and **no** `concept`, and
  `result = true`. "There is definitively no mapping" is an answer, not a failure
* `unmapped` answers the question "this map covers the code, but has no element for it". A code that
  is outside the map's `sourceScope` is not covered at all, and is disposed of by the scope rule above
  rather than by `unmapped`
* The server SHALL support `ConceptMap.group.unmapped` in its modes:
  * `fixed`: any code not otherwise mapped translates to the stated code
  * `other-map`: the translation is delegated to the named ConceptMap, and `used-conceptmap` reports it

#### Reverse translation

* **In R5**, the reverse direction is expressed by naming the *target* concept: `targetCode`,
  `targetCoding` or `targetCodeableConcept`. The `reverse` parameter does not exist in R5, and a server
  that receives it SHALL return a 4xx with a `not-supported` issue rather than guessing what was meant
* **In R4**, the `reverse` parameter does exist, and an R4 server SHALL support it, as well as the
  `targetcode` form

### $closure

`$closure` is tested by the `closure` suite, and its requirements are not yet written up here.

### $compare

The `$compare` operation - determining whether two value sets are equivalent, or one a subset of the
other, or they overlap, or they are disjoint - is being trialled on tx.fhir.org. It is **not** a
requirement for registration in the ecosystem, and its tests are gated behind the `tx.fhir.org` mode.
This section will be written when the operation is settled.

### Code System Versions

*(suites: `version`, `overload`, `default-valueset-version`, `valueset-version`)*

Version handling is the largest single body of tests in the suite, because it is where servers most
often differ. The rules:

* Where the request or the value set names a version, that version SHALL be used
* Where neither does, the server uses its own default, and SHALL report which version it used - in
  `used-codesystem` for `$expand`, and the `version` parameter for `$validate-code`
* `system-version` supplies a **default**. Anything the value set states explicitly overrides it
* `force-system-version` **overrides** the value set
* `check-system-version` SHALL produce an error where the value set asks for a different version
* Validating a code against a version it does not exist in SHALL fail, even where the code exists in
  another version of the same code system
* Validating a display against the wrong version SHALL fail in the same way: displays are
  version-specific
* A CodeableConcept may carry codings from more than one version of a code system, and that is not in
  itself an error

### Inactive, Deprecated and Not-Selectable Codes

Code systems vary in whether codes and/or designations can be labelled as inactive, and if they do, how it is done. 
SNOMED CT defines 'inactive' explicitly. For other Code Systems, codes or designations are labelled as 'should not use' 
in any fashion, they are regard as inactive.

For the CodeSystem resource:
- a concept that has a 'http://hl7.org/fhir/concept-properties#status' property value of 'inactive' or 'retired' is inactive
- a status of 'deprecated' does not make a concept inactive - the concept is still valid to use, though its use is discouraged
- if there is no status property for a concept, the standards-status extension (see below) may provide the status
- for a designation, the only way to denote inactive (at this time) is to use the standards-status extension.

If a concept is defined as inactive:
* inactive SHALL be true when the concept is found in an expansion 
* in expansions, the status property SHALL be populated with a status indicating why the concept is inactive (usually 'inactive', 'retired', or 'withdrawn'). This is not optional - a server that knows a concept is inactive SHALL say why
* the return value from $validate-code for the concept SHALL include a warning that the code is invalid, and the parameters 'inactive' and 'status' SHALL be populated

If a designation is defined as inactive:
* if the designation is included in an expansion, it SHALL either have a standards-status extension with a value of either 'withdrawn' or 'deprecated', or a use code of 'http://snomed.org/info#900000000000546006'
* if provided as the display to $validate-code, an inactive display causes a warning that the display is no longer current (but it is valid)

A concept may also be deprecated or withdrawn *by the value set* rather than by the code system, using
the `valueset-deprecated` or `standards-status` extension on `ValueSet.compose.include.concept`. A
server SHALL honour that too, and report it in the same way *(suite: `deprecated`)*.

Not-selectable is a different thing again: the code exists and is valid, but it is not meant to be used
in an instance *(suite: `notSelectable`)*.

* A concept is not selectable where it carries a property whose canonical uri is
  `http://hl7.org/fhir/concept-properties#notSelectable` with the value true - see
  [Concept Properties](#concept-properties) for how that property is recognised
* `notSelectable` SHALL be usable as a filter, with both the `=` and the `in` operators
* In an expansion, a not-selectable concept SHALL be marked `abstract`
* `$validate-code` SHALL accept a not-selectable code only where the request says it will take one:
  with `abstract = true` the code validates, and without it the server SHALL report that the code is
  not selectable
* Where the code system says nothing about whether a concept is selectable, the server SHALL NOT assume
  either answer

### Extensions

The following extensions SHALL be supported:

* `http://hl7.org/fhir/StructureDefinition/codesystem-alternate` - if code system has alternate codes (TODO: this is subject to further discussion)
* `http://hl7.org/fhir/StructureDefinition/codesystem-conceptOrder` - if code system has order, then this SHOULD be echoed (nothing else needed)
* `http://hl7.org/fhir/StructureDefinition/codesystem-label` - if code system supports 'labels', then this SHOULD be echoed (nothing else needed)
* `http://hl7.org/fhir/StructureDefinition/coding-sctdescid` - if sct is in scope (exact use cases need discussion)
* `http://hl7.org/fhir/StructureDefinition/itemWeight` - echo in value set if defined in code system or value set
* `http://hl7.org/fhir/StructureDefinition/rendering-style` -  echo in value set if defined in code system or value set
* `http://hl7.org/fhir/StructureDefinition/rendering-xhtml` -  echo in value set if defined in code system or value set
* `http://hl7.org/fhir/StructureDefinition/valueset-concept-definition` - populate if requested in expansion request
* `http://hl7.org/fhir/StructureDefinition/valueset-deprecated` - populate in the response if code system concept is deprecated
* `http://hl7.org/fhir/StructureDefinition/valueset-supplement` - check for this, blow up if supplement is properly supported
* `http://hl7.org/fhir/StructureDefinition/valueset-label` - echo in value set if defined in code system or value set
* `http://hl7.org/fhir/StructureDefinition/valueset-conceptOrder` - echo in value set if defined in code system or value set
* `http://hl7.org/fhir/StructureDefinition/structuredefinition-standards-status` - may be found on either a concept or a concept designation. The status codes 'withdrawn' and 'deprecated' mean that the concept / designation is inactive. In addition, it might be found on a ValueSet.compose.include.concept to indicate that the concept's inclusion in the value set is deprecated/withdrawn etc

Note that some of these extensions may be supported by rejecting instances that contain them, depending on the 
specific use cases that the server supports. E.g., if the server does not support externally derived code systems 
then the code system extensions are not relevant.

In R4, several R5 elements are carried as cross-version extensions. Those are not optional either - see
[R4 and the Test Cases](r4.html), and `ValueSet.compose.property` above.

### Code System Specific Requirements

The requirements above apply to every server. The requirements below apply only to a server that
supports the code system in question, and each corresponds to a [mode](testcases.html#modes) in the
test runner. A server declares which of these apply to it by the modes it asks to be tested in; a
server that claims a mode SHALL pass that mode's tests.

#### SNOMED CT

*(mode: `snomed`; suites: `snomed`, `sct-ecl`)*

The SNOMED tests run against a fixed test subontology, not against a real edition: it is published in
the `tx-source` directory of this IG's repository, under the edition
`http://snomed.info/xsct/31000003106`. A server that claims the `snomed` mode SHALL load it, because
the tests assert specific concepts, displays and counts that only that subontology has.

* The server SHALL support the [implicit value sets](ecosystem.html#snomed-ct-implicit-valuesets):
  `?fhir_vs` (all of SNOMED), `?fhir_vs=isa/{sctid}`, `?fhir_vs=refset/{sctid}` and
  `?fhir_vs=ecl/{expression}`. Where the implicit value set names a refset or a concept that does not
  exist, the server SHALL return an error naming the value set that was asked for
* The server SHALL support the `concept` filter with `is-a`, `descendent-of`, `is-not-a` and
  `child-of`, and the SNOMED-specific `expressions` filter
* The server SHALL support post-coordinated expressions, and SHALL observe whether the value set allows
  them:
  * where the value set says `expressions = false`, a well-formed expression SHALL be rejected as not
    in the value set, even where its focus concept is in the value set
  * where the value set says `expressions = true`, a well-formed expression whose focus concept is in
    the value set SHALL be accepted
  * where the value set enumerates codes, an expression is in the value set only where the expression
    *itself* is enumerated. Enumerating its focus concept does not admit it
* Expressions SHALL be parsed according to the SNOMED CT compositional grammar (SCG). In particular the
  comma before an attribute group is optional, so an attribute set followed directly by a group SHALL
  be accepted, and normalises to the form with the comma
* `$validate-code` on an expression SHALL check the expression against the machine-readable concept
  model (MRCM) where the server has it, and report a violation as an error: an attribute outside its
  domain, a value outside its range, an attribute used more times than its cardinality allows, an
  ungrouped attribute written inside a relationship group or vice versa, laterality on a body structure
  that is not in the lateralizable reference set, and the concrete-value rules (a concrete value where
  a concept is required, or the reverse, and concrete values outside their stated range)
* An attribute with more than one domain is valid in any of them
* Where the server cannot fully check an expression, it SHALL say so with a `processing-note` issue
  rather than silently passing it
* `$subsumes` SHALL work between expressions, and between an expression and a precoordinated concept,
  in both directions. Where the structure of the two expressions does not settle the question - they
  refine different attributes, or the same attribute with unrelated values - the server SHALL return an
  error saying it cannot determine the relationship. Finding no relationship is not the same as
  establishing that there is none, and the server SHALL NOT report `not-subsumed` in that case
* `$subsumes` SHALL reject an expression that is not valid under the MRCM rather than answering it
* The server SHALL return the display in the requested language, following the edition's language
  reference sets - see [Languages](languages.html)

#### LOINC

*(mode: `tx.fhir.org` at present; suite: `tx.fhir.org`)*

LOINC requirements are currently tested only against tx.fhir.org, because they depend on which LOINC
release and accessory files the server has loaded. A server that wants to be tested for LOINC should
contact the FHIR product director. The behaviour tested covers the LOINC properties (`COMPONENT`,
`METHOD_TYP`, `ORDER_OBS`, `SCALE_TYP`, `CLASS`), `STATUS`, the parent/child hierarchy, answer lists,
parts, and the `answers-for` and `list` filters.

#### UCUM

*(mode: `tx.fhir.org` at present; suite: `UCUM`)*

* The server SHALL validate UCUM codes by the UCUM grammar, not by a list
* The server SHALL support the two special value sets: all UCUM codes, and the canonical units. Note
  that the URL of the 'all codes' value set changed between R4 and R5
* `$lookup` SHALL work for a UCUM expression, including one carrying an annotation
* `$subsumes` on UCUM SHALL report `equivalent` for two spellings of the same unit - a named derived
  unit and the expression it is defined as, a unit with and without an annotation - and
  `not-subsumed` otherwise. UCUM has no hierarchy, so a scaled unit is not subsumed by the unit it
  scales, two units sharing a canonical unit but differing in magnitude are not related, and neither
  are two dimensionless units that differ only in magnitude
* An invalid UCUM code in a `$subsumes` request SHALL produce an error

#### Mime Types (BCP 13)

*(mode: `mimetypes`; suite: `mimetypes`)*

Media types have no hierarchy between types and subtypes, but parameters narrow a media type, and that
is what makes subsumption meaningful.

* The server SHALL support the `base` filter, selecting by type (`text`) or by type and subtype
  (`text/plain`). Parameters on a code SHALL NOT stop it matching a base filter
* The server SHALL support the `registered` filter, selecting media types that are or are not in the
  IANA registry
* The code system cannot be expanded as a whole, and neither can a `base` filter or `registered=false`:
  those are unbounded, and the server SHALL answer `too-costly` rather than returning a partial
  expansion presented as complete. `registered=true` can be expanded, narrowed by a base filter, and
  SHALL be marked as an unclosed expansion
* `$subsumes` SHALL treat a media type carrying a parameter as subsumed by the same type without it,
  and a parameter set as subsumed by a superset of it. Media types are case-insensitive
* Where a parameter has a default defined by its RFC, the bare form already carries that default. So
  `text/plain` and `text/plain; charset=us-ascii` are `equivalent`, and `text/plain` and
  `text/plain; charset=utf-8` are `not-subsumed` - the parameter contradicts the default rather than
  narrowing it
* Where the server does not know a parameter, it cannot know whether the parameter narrows the type, and
  SHALL return an error rather than guessing - unless both codes carry the parameter identically, in
  which case it cannot affect the answer and SHALL NOT stop the server deciding

#### IETF Language Codes (BCP 47)

*(mode: `tx.fhir.org` at present; suite: `langcodes`)*

* The server SHALL validate tags against the IANA subtag registry, including the registry's Prefix
  rules: an extlang or variant may only follow the prefix its registration names
* Private-use ranges written in the registry as ranges (`QM..QZ`) SHALL be accepted
* Grandfathered tags are registered whole, and SHALL be validated whole - they do not decompose into
  subtags
* Tags are case-insensitive: a tag in the wrong case is valid, and the server SHALL return the
  canonical casing in `normalized-code` and note the difference
* `$subsumes` SHALL implement RFC 4647 extended filtering: a tag subsumes any tag that adds subtags to
  it, including where the added subtag sits between two the shorter tag names (`en-US` subsumes
  `en-Latn-US`). A grandfathered tag subsumes nothing
* The server SHALL support the `language`, `region` and `script` filters. An absent component does not
  match a fixed one: a tag with no region is not in a `region=US` value set, and a tag with no script
  is not in a `script=Latn` value set, even where that is the script the language is written in
* A `language` filter is finite and SHALL be expandable (paged). A `region` filter alone is finite but
  far too large, and a `script` filter alone leaves the language open: both SHALL answer `too-costly`.
  Expanding all language codes SHALL return the common-languages base value set, marked as unclosed

#### OMOP

*(mode: `omop`; suite: `omop`)*

The OMOP tests are based on a stable subset maintained for the ecosystem. Some servers support only
OMOP, and the tests are written so that they can.

* The server SHALL validate OMOP codes as `code`+`system`, `Coding` and `CodeableConcept`, with and
  without a value set, and SHALL check both the display and the version
* `$lookup` SHALL work for standard OMOP concepts. A non-standard concept may be looked up or refused,
  depending on the server
* The server SHALL support value sets filtered by OMOP domain
* Translating from OMOP to LOINC without naming a ConceptMap is optional - a server that is not willing
  to choose a map on the client's behalf is not required to

#### ICD-11

*(mode: `icd-11`; suite: `icd-11`)*

The ICD-11 tests assert correct FHIR behaviour rather than the behaviour of any particular server, and
`tests/icd-11/doco.txt` in this IG's repository explains each one. Three code systems are in play, each
with its own canonical: the Foundation (`http://id.who.int/icd/entity`), the MMS linearization
(`http://id.who.int/icd/release/11/mms`) and ICF (`http://id.who.int/icd/release/11/icf`).

* The server SHALL support `$lookup` and `$validate-code` against each of the three, by code and by
  entity URI, including grouper concepts and residual categories
* `$lookup` SHALL echo `code` as it was asked for. A request by entity URI gets the entity URI back;
  the normalised short form belongs in the `code` property
* The server SHALL support ICD-11 post-coordination in both `$lookup` and `$validate-code`, including
  clustered and repeated axis values, and SHALL reject an invalid axis
* The server SHALL support expansion of the post-coordination scales, with `count`, `offset` and
  `filter`
* The server SHALL support language, and reject a language it does not have
