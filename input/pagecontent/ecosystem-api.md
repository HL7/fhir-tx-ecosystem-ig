The [monitoring server](ecosystem-monitor.html) provides a public API that has three sub-functions: 

* Discovery: list known servers 
* Resolution: recommend which server to use for a code system
* Server Choice: choose the one server to use for a request that involves several code systems (optional)

The ```root``` of this API is the address of the coordination server, which the ecosystem's own documentation provides.

### Discovery 

```GET {root}```

§The standard discovery parameters SHALL be supported by all servers§:

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
  "master-url": "https://example.org/tx-servers.json", // master registry that was scanned
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
    "authoritative": [], // the CodeSystem masks the server claims authority for (as in the registration)
    "authoritative-valuesets": [], // the ValueSet masks the server claims authority for (as in the registration)
    "candidate": [], // when url is given: [url] if the endpoint hosts it without claiming authority
    "open": true // if the server supports non-authenticated use 
    "password" | "token" | "oauth" | "smart" | "cert": true 
       // if the server supports authentication by one or more of those methods
  }]
}
```

Notes:

* server-*X* and registry-*X* properties are provided for human trouble-shooting; they are not used anywhere
* When `url` is given, only endpoints that host that code system are listed, those authoritative for it first
* Without `url`, there is no candidate list: listing everything each server hosts would make the response very large

### Resolution

A client can also ask which server to use for a particular CodeSystem or ValueSet. 

```GET {root}/resolve?fhirVersion={ver}&url={url}```

§The standard resolution parameters SHALL be supported by all servers§:

* `fhirVersion` (**required**): return only those endpoints that are based on the given FHIR version (RX or M.n.p)
* `url`: return only those endpoints that support a particular code system (by canonical, so url or url|version)
* `valueSet`: return only those endpoints that know a particular ValueSet (by canonical, so url or url|version)
* One of `url` or `valueSet` must be present
* `authoritativeOnly`: return only those endpoints that are authoritative (true or false; default is false)
* `language`: the language of the request being routed. Accept-Language syntax: a single BCP-47 tag, or a 
  weighted list (e.g. `de-AT, de;q=0.9, en;q=0.1`) for clients that want their fallback policy expressed 
  in a single call. Optional
* `usage` - see below

Clients should pass `language` when (and only when) the operation being routed is language-sensitive: 
an `$expand` where a display language is in play, a `$validate-code` that carries a display language 
(whether or not a display is provided to check - the display in the response comes back in the 
requested language), a `$lookup` requesting designations. Requests with no language interest should 
omit it. The value passed is whatever the client resolved from the displayLanguage sources described 
in [Languages](languages.html) - resolution and the subsequent operation must use the same language, 
or the routing is meaningless.

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
* A server that hosts any version of a SNOMED CT edition hosts that edition: a request for 
  `http://snomed.info/sct|http://snomed.info/sct/32506021000036107` matches a server that hosts 
  `http://snomed.info/sct|http://snomed.info/sct/32506021000036107/version/20250331` 
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


### Server Choice

Resolution answers for one code system (or value set) at a time, but a single terminology request 
often involves several: a value set that includes LOINC and a national code system, or a 
CodeableConcept with codings from three different code systems. The client has to send the 
request to one server, so it has to combine the resolution answers into a single choice. 

The HAPI core Java tools make this choice client side. Server Choice makes the same choice on 
the coordination server, so that other clients can get the same routing without reproducing 
the logic. It doesn't necessarily replace the client side logic - clients are free to make their own choice 
from the resolution answers, but it is more convenient to let the server choose, and offers the 
administrator a single place to tune their ecosystem's behaviour.

This function is required for coordination servers - they are required to support it.

```GET {root}/choose-server?fhirVersion={ver}&default={server}&system={url1}&system={url2}```

Parameters:

* `fhirVersion` (**required**): as for resolution
* `default` (**required**): the client's default terminology server - the server it would use without 
  the ecosystem. It is the fallback when nothing else decides, and the server used for the code 
  systems that get special handling (see below)
* `system`: a code system that the request involves, by canonical (url or url|version). Repeat the 
  parameter for each code system. If there are none, the answer is the default server
* `valueSet`: the value set the request is about, by canonical, if there is one. It is used for 
  logging, and as the last fallback (below)
* `language`: as for resolution, and under the same rules: only for language sensitive requests
* `usage`: as for resolution

Before asking, the client has to work out the code systems the request involves. This is client 
side work, because the coordination server can't do it - the content it needs is often only 
available to the client (e.g. an IG being built, a set of valuesets defined by the client application):

* the code systems of the codings, and of the value set's include and exclude statements, following 
  any imported value sets. If the client can't resolve an imported value set, it can't know the 
  code systems involved, and it should use its default server without asking
* any version that the expansion parameters fix (`system-version`, `force-system-version`) applied 
  to the code system canonical
* for an expansion, the `displayLanguage` from the expansion parameters as the language, if no other 
  language applies

The return value is a JSON object:

```json5
{
  "server" : "http://server/endpoint", // the FHIR endpoint to send the request to
  "log" : [ // how the server was chosen - for humans (trouble-shooting), not for processing
    "https://healthterminologies.gov.au/fhir/CodeSystem/x: authoritative [https://tx.ontoserver.csiro.au/fhir], candidates []",
    "http://terminology.hl7.org/CodeSystem/v2-0203: authoritative [http://tx.fhir.org/r4], candidates [https://tx.ontoserver.csiro.au/fhir]",
    "Found partially authoritative server https://tx.ontoserver.csiro.au/fhir"
  ]
}
```

The server is chosen using the resolution answer (both authoritative and candidates) for each code 
system, applying these rules in order:

1. a server that is authoritative for all the code systems
2. a server that is authoritative for one of the code systems, and authoritative or a candidate for 
   each of the others
3. code systems that get special handling: 
   * if the request involves several code systems including SNOMED CT, the choice made for SNOMED CT 
     alone (i.e. SNOMED CT decides)
   * if the request involves a code system that always goes to the primary server, the default server. 
     In the HL7 ecosystem, these are the code systems that tx.fhir.org handles specially (UCUM, LOINC, 
     SNOMED CT, RxNorm, CVX, NDC, CPT, BCP-13, BCP-47, RFC 3986, ISO 3166, ISO 4217, M49, HGVS and a few others)
4. a server that is a candidate for all the code systems, and that has a usable copy of each of them - 
   one where the content mode the server reports is `complete` or `fragment`, or where the server 
   doesn't report a content mode
5. the server that is authoritative for the most code systems (the first, if there's a tie)
6. the server that the value set comes from (by resolving the `valueSet` parameter), if there is one, 
   else the default server

In rules 2 and 4, a code system that no server is known to host doesn't count against a server.

Notes:

* The special handling in rule 3 reflects the operational arrangements of the HL7 ecosystem, and is 
  under review: it may become part of the ecosystem's registration information in the future
* The client side logic in the HAPI core Java tools follows the same rules, but differs in some 
  details: it checks for a usable copy (rule 4) by searching the candidate server, and it uses the 
  server the value set was actually retrieved from in rule 6


### Usages

If a server is marked as having a restricted use, the server will only be returned in the resolve call above if the client provides a use token that matches one of the tokens in the server's usage property (if it has any).

A client can provide a usage token with the parameter 'usage': ```GET {root}/resolve?fhirVersion={ver}&url={url}&usage=publication```

It's up to the client to provide a usage token that accurately represents its usage. 

The open source HAPI core Java tools that support the ecosystem populate usage with one of three values:
* ```publication``` - the tool is publishing an IG, or building the core FHIR Specification
* ```validation``` - the tool is validating the content of a resource (this may be in production or from the command line, or validator.fhir.org)
* ```code-generation``` - the tool is generating some kind of code

The primary purpose of the usage flag is so that an administrator can deny access to server for purposes of validation. This *might* be appropriate if the ecosystem is not a production grade system (e.g. tx.fhir.org) but there is concern that some users won't restrict themselves from using it operationally (which is not supported by tx.fhir.org) for budget reasons.

