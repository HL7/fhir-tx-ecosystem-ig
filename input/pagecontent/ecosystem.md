This page documents the HL7 terminology ecosystem framework, where a set of servers
that serve different code systems on behalf of different code system authorities, and
there is a central coordinating server that routes terminology operations to the 
correct server. 

A terminology server ecosystem is defined by two URLs:
* the URL of the master registry that registers the servers in the ecosystem
* the URL of the coordination server that handles value set and code system resolution for the ecosystem 

There must be at least one coordination server for the ecosystem, though may be more than one. 

HL7 manages one ecosystem:

* Master registration file: <https://fhir.github.io/ig-registry/tx-servers.json>
* Primary coordination server: <http://tx.fhir.org/tx-reg>

HL7 requires that all servers registered with the HL7 ecosystem have passed the tests described 
in this document as a representation of the requirements documented in this IG. 

Other ecosystems may be defined by other organizations by creating their own pair of registration file 
and server. Individual servers might participate in multiple different ecosystems.

Note that other ecosystems may not make the same requirement about passing these tests; users of the ecosystem 
will have to consult the documentation to determine the applicable registration policy.

### Registering the Servers 

Registration is a two level process:
* The master registration file lists one or more server registries
* The server registries list one or more servers

The files may be static json files (as the HL7 ecosystem practices), or 
they may be dynamically generated in whatever fashion the manager of the 
file sees fit. 

#### The master registration file 

The master registration file has the following content:

```json5
{
  "formatVersion" : "1", // fixed value. Mandatory
  "description" : "some description of this registry set", // purpose etc. Optional
  "documentation" : "See https://build.fhir.org/ig/HL7/fhir-tx-ecosystem-ig/ecosystem.html", // recommended reference to this page. Optional
  "registries" : [{ // list of the registries. Must be present, but may be empty
    "code" : "code",  // code that identifies the registry. A persistent identifier. Alphanumerics only, no spaces. Mandatory
    "name" : "Name",  // human readable name. Can change, can contain any characters (including html type control chars). Mandatory
    "authority" : "Organization name", // name of publishing org. Optional
    "url" : "https://somwhere/path" // actual location of registry. Mandatory
  }]
}
```

Notes:

* The formatVersion number is 1. This might be changed in the future if breaking changes are made to the format
* Each of the entries in the list of registries points to a json document as documented in the next section
* The code for the registry should be stable, since it may be used to identify the set of servers in the Coordination API (see below)

#### The server registries

This file contains a list of terminology servers.

```json5
{
  "formatVersion" : "1",// fixed value. Mandatory
  "description" : "something", // purpose etc. Optional
  "servers" : [{ // list of actual servers
    "code" : "code",  // code that identifies the server. Persistent identifier. Alphanumerics only, no spaces. Mandatory
    "name" : "Name",  // human readable name. Can change; can contain any characters (including html type control chars). Mandatory
    "access_info" : "documentation", // human readable markdown explaining how to get access to the server. Optional
    "url" : "http://server/page", // human landing page for the server. Optional
    "usage" : [ // if present, a list of string usage tags for the intended use of the server. Missing means any use is appropriate. See below for uses. Optional
    ],
    "languages" : { 
      // authoritative claims for particular languages - see Language Specific Claims below. Optional
      // Property names are BCP-47 language tags; a tag covers all more specific tags (de covers de-AT etc.)
      // Property values are mask lists with the same syntax and meaning as authoritative, 
      // applying to requests in that language
      "de" : ["http://domain/*"],
      "de-AT" : ["http://domain/other/*"]
    },
    "authoritative" : [ 
      // a list of CodeSystems the server claims to be authoritative for (see below). Optional
      "http://domain/*", // simple mask, * is a wildcard  
      "http://domain/*|1.0.0", // (may include version masks)
      "http://domain/*|1.*" 
    ],
    "authoritative-valuesets" : [ 
      // a list of ValueSets the server claims to be authoritative for (see below). Optional
      "http://domain/*", // simple mask, * is a wildcard
      "http://domain/*|1.0.0",
      "http://domain/*|1.*" 
    ],
    "exclusions" : [ 
      // a list of CodeSystems and ValueSets to hide from the ecosystem entirely (see Exclusions below). Optional
      "http://domain/*", // simple mask, * is a wildcard
      "http://domain/*|1.0.0",
      "http://domain/*|1.*" 
    ],
    "fhirVersions" : [{ // list of actual endpoint by FHIR version
      "version" : "R4", // can be R(X)
      "url" : "http://server/endpoint" // actual FHIR endpoint of the server for that version
    }]
  }]
}
```

Notes:

* The formatVersion number is 1. This might be changed in the future if breaking or significant changes are made to the format
* The ```languages``` property is a later, backwards compatible addition to the format. Readers that do not know it simply do not see the language specific claims, and route language requests to the general authoritative server - which is exactly the pre-language behavior (see Language Specific Claims below)
* The code for the server should be stable, since it may be used to identify the server in the Coordination API (see below)
* Most servers will only have one endpoint, that is, they will only support one FHIR version, but some support more than one
* If the different versions have different authoritative lists, that's really different servers
* Server endpoints must support FHIR R3 or later. Valid values for version: R3, R4, R4B, R5, R6

#### Conformance Requirements 

All endpoints listed in the server registration file shall return CapabilityStatement and TerminologyCapabilities
resources that conform to the requirements described in this IG

The HL7 ecosystem requires that all servers fully conform to the requirements laid out in
this implementation, and that servers pass the set of tests described here in, or an acceptable 
alternative set of tests as agreed with the FHIR Product Director (for servers that only support 
specific code systems).

Servers may require authentication that restricts their use to an approved subset of users.

#### Authoritative Servers 

Servers may be declared to be authoritative for a set of CodeSystems and/or ValueSets in the registration file. 
This is a declaration that the server manager believes that the server is the appropriate server in this 
ecosystem to choose for that set of CodeSystems and ValueSets when they are used. 

In general, users will expect that the authoritative servers have the latest versions of the content,
and that older versions still in use are also maintained. The authoritative claim is only in regard 
to the ecosystem, not an absolute claim that the server is the only authoritative server in all 
contexts.

Note that servers often support multiple code systems for which they are not regard as authoritative e.g. tx.fhir.org
has a copy of the Australian edition of SNOMED CT, but the HL7 Australia official server is the authoritative server
for the Australian edition. Servers are not restricted to only serving the content for which they are authoritative.

The authoritative flag is used to help resolve which server to use - see below.

#### Language Specific Claims

Claims in the `authoritative` list are not language specific: the server is the appropriate 
destination for the matched code systems whatever language a request asks for (or none), and it 
responds with whatever languages it actually has, per the usual fallback rules described in 
[Languages](languages.html). This is not an assertion that the server can supply displays in every 
language - it is simply the default destination for the code system.

Sometimes, though, a different server is the appropriate destination for particular languages of 
the same code system, because it has different supplements loaded. For example, one server is the 
general authoritative server for LOINC, while a different server hosts LOINC with the German 
language pack loaded, and is the appropriate server for German-language requests.

The `languages` property makes claims for particular languages: each property name is a BCP-47 
language tag, and its value is a list of masks with the same syntax and meaning as `authoritative`, 
applying to requests in that language. A single server entry can carry both kinds of claim:

```json5
"authoritative" : ["http://fhir.de/CodeSystem/*"], // authoritative, whatever language is asked for
"languages" : {
  "de" : ["http://loinc.org*"] // authoritative for German-language requests
}
```

This entry means: this server is the authoritative server for the `http://fhir.de/` code systems 
whatever language is asked for, and *for German-language requests*, it is also the authoritative 
server for LOINC.

Note that a language specific claim is full authority over the whole operation for requests in that 
language - it is not authority over displays that is separable from authority over content. There is 
no such separation in the API: a `$validate-code` with a German display, or an `$expand` with 
`displayLanguage: de`, is a single indivisible operation that is answered by a single server. 
Consequences:

* A server making language specific claims SHALL host the content of the claimed code systems 
  (`complete` or `fragment`) as well as the relevant supplement(s), so that it can execute the 
  operations outright. Registering a server that holds only supplements is not supported: routing 
  part of an operation to a supplement-only server would require cross-server composition (content 
  from one server + supplement from another via `tx-resource`), and the ecosystem does not 
  (currently) provide this
* Language specific claims are invisible to requests that do not specify a language - they never 
  bleed into general traffic. A request with no language matches only `authoritative` claims, which 
  preserves the existing (pre-language) behavior exactly
* The `authoritative` list remains the default destination: it applies whatever language is asked 
  for (or none), except where another server makes a more specific claim for the requested 
  language - exactly as before

Language specific claims apply to code systems only; there is (at this time) no language dimension 
to `authoritative-valuesets`.

Whether a server actually honours its language specific claims (returning correct displays and 
designations in the claimed languages for the claimed code systems) is verified by the ecosystem 
test cases, not by the coordination server - the coordination server only verifies that the server 
hosts the code systems themselves.

##### Language Matching

BCP-47 language tags are hierarchical, so a language tag is inherently a mask: a claimed tag matches 
any request language that is equal to it, or that is more specific than it (matching on `-` 
boundaries). A claim for `de` matches requests for `de`, `de-AT`, and `de-CH-1996` (but not `dent`); 
a claim for `de-AT` matches `de-AT` but not plain `de`. 

Where more than one server has a matching language specific claim, the most specific matching tag 
wins. Servers matched on their `authoritative` claims rank after all language specific matches (when 
the request specifies a language).

Where the request language is a weighted list per the Accept-Language syntax (e.g. 
`de-AT, de;q=0.9, en;q=0.1`), this matching is applied per language in descending weight order.

#### Exclusions

Servers may hold code systems or value sets that should not be used through the ecosystem at all - 
for example, a copy of a code system that is incomplete, out of date, or not intended for use 
beyond the server's own community. The `exclusions` list hides such content: anything matching one 
of the masks is removed from the server's presence in the ecosystem entirely:

* The coordination server ignores matching code systems and value sets when scanning the server
* The server is never returned by the Resolution API for a matching code system or value set - 
  neither as authoritative nor as a candidate - even if one of the server's authoritative claims 
  (including language specific claims) would otherwise match it

An exclusion is the opposite of an authoritative claim: an authoritative claim says 'prefer this 
server for this content', an exclusion says 'never use this server for this content' (within this 
ecosystem). Note that exclusions are about the server's fitness to serve the content, not about the 
authority to serve it; a server that simply doesn't want to be the *preferred* destination for 
something should instead write its authoritative masks so that they don't cover it.

### The Coordination server

The Coordination server keeps a current set of information about the servers in its
ecosystem, and users of the ecosystem ask the Coordination server which terminology server should be
used for a particular operation. 

#### Scanning the Ecosystem

The Coordination server scans the servers referenced from the master registration 
file on a regular basis, and maintains a live list of the servers, their configuration, and what 
CodeSystems and ValueSet each endpoint supports and is authoritative for. The frequency of scanning 
is a matter for the ecosystem and server administrators, but is generally every few minutes.

The Coordination server will use the ```/metadata``` and ```/metadata?mode=terminology```
operations, and also iterate the full set of ValueSets on the server using ```/ValueSet?_summary=true```.
Note that the coordination server does not search the CodeSystems; instead, the list of 
supported CodeSystems comes from the TerminologyCapabilities resource.

If the server requires authentication for these operations, the Coordination server
must be given access to these operations. 

#### The Coordination server API

The monitoring server provides a public API that has two sub-functions: 

* Discovery: list known servers 
* Resolution: recommend which server to use for a code system

For the HL7 ecosystem, the ```root``` of this API is at [http://tx.fhir.org/tx-reg/](http://tx.fhir.org/tx-reg/)

#### Discovery 

```GET {root}```

These parameters SHALL be supported by all servers:

* `registry`: return only those endpoints that come from a nominated registry (by the code in the master registration file)
* `server`: return only those endpoints that have the code given
* `fhirVersion`: return only those endpoints that are based on the given FHIR version (RX or M.n.p)
* `url`: return only those endpoints that support a particular code system (by canonical, so url or url|version).
* `authoritativeOnly`: return only code systems which the endpoints are authoritative for (true or false; default is false)
* `language`: filter authoritative claim matching by language (only meaningful in combination with `url`; see Language Specific Claims above)

When the ```Accept``` header is ```application/json```, the return value is a JSON object:

```json5
{
  "last-update": "2023-07-24T04:12:07.710Z", // last time the registries were scanned
  "master-url": "https://fhir.github.io/ig-registry/tx-servers.json", // master registry that was scanned
  "results": [{ // list of discovered endpoints
    "server-name": "human readable name",
    "server-code": "human readable name",
    "registry-name": "HL7 Terminology Services",
    "registry-code": "persistent code for the registry",
    "registry-url": "http://somewhere",
    "url": "http://server/endpoint", // actual endpoint for the server
    "fhirVersion": "4.0.1", // FHIR version - semver
    "error": "string", // details of error last time server was scanned, or null
    "last-success": int, // number of milliseconds since the server was last seen up
    "systems": int, // number of code systems found on the server
    "authoritative": [], // list of authoritative CodeSystems as canonical values (url|version)
    "authoritative-valuesets": [], // list of authoritative ValueSets as canonical values (url|version)
    "candidate": [], // list of candidate CodeSystems as canonical values (url|version)
    "candidate-valuesets": [], // list of candidate ValueSets as canonical values (url|version)
    "open": true // if the server supports non-authenticated use 
    "password" | "token" | "oauth" | "smart" | "cert": true 
       // if the server supports authentication by one or more of those methods
  }]
}
```

Notes:

* server-*X* and registry-*X* properties are provided for human trouble-shooting; they are not used anywhere

#### Resolution

A client can also ask which server to use for a particular CodeSystem or ValueSet. 

```GET {root}/resolve?fhirVersion={ver}&url={url}```

These parameters SHALL be supported by all servers:

* `fhirVersion` (**required**): return only those endpoints that are based on the given FHIR version (RX or M.n.p)
* `url`: return only those endpoints that support a particular code system (by canonical, so url or url/version)
* `valueSet`: return only those endpoints that know a particular ValueSet (by canonical, so url or url/version)
* One of `url` or `valueSet` must be present
* `authoritativeOnly`: return only those endpoints that are authoritative (true or false; default is false)
* `language`: the language of the request being routed. Accept-Language syntax: a single BCP-47 tag, or a 
  weighted list (e.g. `de-AT, de;q=0.9, en;q=0.1`) for clients that want their fallback policy expressed 
  in a single call. Optional
* `usage` - see below

Clients should pass `language` when (and only when) the operation being routed is language-sensitive: 
an `$expand` where a display language is in play, a `$validate-code` where a display will be checked, 
a `$lookup` requesting designations. Pure code validation should omit it. The value passed is whatever 
the client resolved from the displayLanguage sources described in [Languages](languages.html) - 
resolution and the subsequent operation must use the same language, or the routing is meaningless.

When `language` is present, servers with matching language specific claims are returned first in the 
`authoritative` list (ordered by the language matching rules above, then by the usual ranking), 
followed by servers matched on their `authoritative` claims. When `language` is absent - or for 
value set resolution, where the claims have no language dimension - behavior is exactly as it was 
before language support was added.
  
When the ```Accept``` header is ```application/json```, the return value is a JSON object:

```json5
{
  "formatVersion" : version,
  "registry-url" : url,
  "authoritative" : [{
    "server-name" : "Human name for server",
    "url" : "http://server/endpoint", // actual FHIR endpoint of the server 
    "fhirVersion" : "4.0.1", // FHIR version - semver
    "open" | "password" | "token" | "oauth" | "smart | "cert": true, // as above
    "access_info" : "description", // provided if the server has some, for error messages
    "languages" : ["de"] // if this entry was matched on a language specific claim: the language 
                         // tag of that claim
  }],
  "candidates" : [{
    // same content as for authoritative, except that content will
    // be reported, *if* provided by the server
    "content": "not-present" | "example" | "fragment" | "complete" | "supplement",
    "language-support" : "unknown" // present when the request specified a language: the ecosystem
                                   // does not know whether this server can supply that language (see below)
  }]
}
```

Notes:

* The resolve operation may return more that one server that is labelled as authoritative if the ecosystem is defined that way. Resolving this is up to the client
* The resolve operation may return more than one candidate server if more than one server hosts the terminology. Resolving this is up to the client
* A server listed as authoritative won't also be listed as a candidate
* Servers are not listed as authoritative unless they actually host the CodeSystem(+version) in the request 
* An authoritative entry without a `languages` property was matched on the server's `authoritative` 
  claims rather than a language specific claim. When the request had a language parameter, this tells 
  the client that routing fell through to the default authoritative server - whether to use it anyway 
  or treat the language as unavailable is the client's decision (the same strictness decision as the 
  `de, *;q=0.1` vs `de, *;q=0` distinction described in [Languages](languages.html))
* Candidate servers are not filtered or ranked by language - the ecosystem has no reliable way to know 
  what languages a candidate can serve (that information comes from registration claims, and candidates 
  by definition have none that match). Instead, when the request specifies a language, candidate 
  entries are explicitly marked with `language-support: unknown`. A marked candidate hosts the code 
  system and can execute the operation, but may not be able to supply the requested language: using 
  one means accepting displays in whatever language it has. That is only a valid choice under the 
  wildcard reading of the language (`de, *;q=0.1`); clients that require the requested language 
  (`de, *;q=0`) should not use marked candidates. Note in particular that validating a display 
  against a server that lacks the requested language produces a false negative, not a degraded 
  result - a marked candidate is only a safe fallback for operations where wrong-language displays 
  are acceptable


### Usages

If a server is marked as having a restricted use, the server will only be returned in the resolve call above if the client provides a use token that matches one of the tokens in the server's usage property (if it has any).

A client can provide a usage token with the parameter 'usage': ```GET {root}/resolve?fhirVersion={ver}&url={url}&usage=publication```

It's up to the client to provide a usage token that accurately represents its usage. 

The open source HAPI core Java tools that support the ecosystem populate usage with one of three values:
* ```publication``` - the tool is publishing an IG, or building the core FHIR Specification
* ```validation``` - the tool is validating the content of a resource (this may be in production or from the command line, or validator.fhir.org)
* ```code-generation``` - the tool is generating some kind of code

The primary purpose of the usage flag is so that an administrator can deny access to server for purposes of validation. This *might* be appropriate if the ecosystem is not a production grade system (e.g. tx.fhir.org) but there is concern that some users won't restrict themselves from using it operationally (which is not supported by tx.fhir.org) for budget reasons.

### SNOMED CT Implicit ValueSets 

There's a limitation of use associated with the ecosystem, which is a little tricky to explain. 

The page [Using SNOMED CT with HL7 Standards](https://terminology.hl7.org/SNOMEDCT.html) defines a series of implicit SNOMED CT value set URLs. These implicit value sets are very useful since profiles, ValueSets, etc., can just refer to them directly without having to ensure that they are created, and they are supported by all the terminology servers in the ecosystem. However there's a limitation on their use associated with the need to decide which SNOMED CT server is the correct one to use for any particular request. 

The implicit value set URL has two parts:

* **base**: the SNOMED CT edition/version on which the value set is based, e.g. `http://snomed.info/sct/20611000087101` is any Canadian version
* **specifier**: the details of the value set, e.g. `?fhir_vs=refset/22071000087104`

Given the way that the value sets are defined, they are only valid against a given edition and version, and can only be evaluated (and even instantiated) against that known version. One way to manage this is to bind the value set to a specific version in the definition itself,  e.g. `http://snomed.info/sct/20611000087101/version/20240731?fhir_vs=refset/22071000087104` identifies a specific version. 

But it's often more useful to allow late binding of the value set to a specific version. E.g. `http://snomed.info/sct/20611000087101?fhir_vs=refset/22071000087104` is whatever is the applicable SNOMED CT version, where that might be specified using a `system-version` parameter in the expansion parameters, or the server might just use the latest version of the Canadian edition that it has. 

In the same way, `http://snomed.info/sct?fhir_vs=refset/22071000087104` means 'whatever version of whatever edition', as decided by the infrastructure when the reference is used. (this particular example neatly shows the risk of this approach, because 22071000087104 is a Canadian specific reference set, but it would be fine for most SCT content).

Unfortunately, the ecosystem infrastructure doesn't know how to route `http://snomed.info/sct?fhir_vs=refset/22071000087104` to the correct server - at the point the decision is made, the actual SNOMED CT edition isn't known, and can't be known, and the correct server can't be picked. Any evaluation of this value set will be done by tx.fhir.org, not one of the designated SNOMED CT servers (in this case, the Canadian server would be desired).

Hence, in the terminology ecosystem, all SNOMED CT implicit value sets must include the edition if they're not based on the international edition, and if evaluation by the correct server is desired. 

This restriction applies to value set references in StructureDefinitions, Questionnaires, ValueSets, and other resources used in IGs, packages, etc. It does not apply to value sets used by import on one of the servers - the ecosystem does not resolve such a value set, and it's up to the server to do so.

Note: this restriction might change in the future e.g. if there's only one SNOMED CT server for all editions, but there's no plan for that at this time. 

### Language and Routing

A similar limitation applies to languages: the router can only act on what is in the resolve call, so 
the language must be known at routing time. Clients resolve their display language (from the request 
parameter, value set extension, HTTP header, or ValueSet.language, per [Languages](languages.html)) 
*before* asking the ecosystem which server to use, and pass that language in the resolve call. 

If the language is not known or not specified at routing time, the client gets default routing (the 
unqualified authoritative server) and the default language behavior of that server - which is the 
correct outcome for a request that never specified a language, but means that a language specified 
only *after* server selection (e.g. decided later in an expansion workflow) will not influence which 
server was chosen.