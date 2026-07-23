### Multi-Language Support 

This page documents the expectations for servers in the terminology ecosystem around language support. 
Servers in the ecosystem must support the minimum features in this page around validation, whether or
not the affected code system has more than one language supported or not. 

Servers are also generally required to support CodeSystem supplements for language packs, though this
may be waived for some code systems with their own alternative translation support and governance on 
discussion with HL7.

#### Client Language Specification

Clients might want to do one of three things:

* just accept whatever language the server is going to return, based on the server default and the underlying code system definitions
* specify the return language desired, and whether or not to fall back to other languages if the desired language isn't available
* specify the return language and in addition, request designations from additional languages (all available, or specified languages)

Clients can specify the return displayLanguage in one of four places:

* Using a displayLanguage parameter in the request
* Using a displayLanguage parameter in the value set using the 
[[[http://hl7.org/fhir/tools/StructureDefinition/valueset-expansion-parameter]]] extension
* Using the http Accept-Language header 
* ValueSet.language

The first three may contain multiple languages with weights per the Accept-Language specification; which to use is a matter 
of operational control on the client. ValueSet Language is a single language code as described in BCP-47. 

* Servers SHALL support all four approaches 

From a server point of view, the order of priority for the displayLanguage parameter is 
`request parameter` > `valueset-extension` > `HTTP Header` > `ValueSet.language`

#### Language and Server Selection

Language doesn't just affect what a server returns - in the [ecosystem](ecosystem.html), it affects 
*which server should be used*, because authoritative claims can be made for particular languages: 
different servers may be the appropriate destination for the same code system depending on the 
language of the request (typically because they have different supplements loaded).

Consequences for clients using the ecosystem:

* Clients SHALL resolve the display language (from the four sources above, in priority order) *before* 
  server selection, and pass it as the `language` parameter in the coordination server's resolve call 
* The language used for routing and the language used in the subsequent operation must be the same
* Only language-sensitive operations should be routed with a language: `$expand` where a display 
  language is in play, `$validate-code` that carries a display language (even when no display is 
  provided to check, the display in the response comes back in the requested language, and callers 
  use it - e.g. display lookups when rendering), `$lookup` requesting designations. In practice: 
  route with the language whenever the request specifies one; requests with no language interest 
  should not invent one
* If no server has a language specific claim matching the language, the resolve call returns the 
  default authoritative server. Whether to use it anyway (accepting displays in whatever language 
  it has - the `de, *;q=0.1` behavior) or to treat the language as unavailable (the `de, *;q=0` 
  behavior) remains the client's decision, exactly as it is for a single server
* Candidate servers returned for a language-specific resolve are marked `language-support: unknown` - 
  they host the code system, but the ecosystem does not know whether they can supply the requested 
  language. They are only usable under the wildcard reading; in particular, do not validate displays 
  against them

See [Language Specific Claims](ecosystem.html#language-specific-claims) for the registration format, 
matching rules, and resolution ordering.

### Value Set Language Control

Value sets may them selves make rules about the language behavior using a combination of:

* Specifying a language in ValueSet.language
* Providing language specific displays and designations on ValueSet.compose.include.concept 
* Specifying a specific displayLanguage parameter in a [[[http://hl7.org/fhir/StructureDefinition/valueset-expansion-parameter]]] extension

### Server Behavior

The correct responses depends on the interplay between these various parameters, and are 
effectively specified by the test cases.

#### Test Cases

the tests here test out these combinations. To test this, we define four code systems, all basically related
to each other:

* codesystem-en-single.json - a set of codes with english displays
* codesystem-de-single.json - the same set of codes with german displays
* codesystem-en-multi.json - the same set of codes with english displays and additional displays and definitions in german with a smattering of other variants
* codesystem-de-multi.json - the same set of codes with german displays and additional displays and definitions in english with a smattering of other variants

Then a set of value sets are defined on this, that include all codes from one of those code systems. Some of the value sets specify the language of expansion (discussed below)

### Simple Echo

The first four tests simply echo the four code systems back with no language rules

* **language-echo-en-none**: just return codes from an english based code system. All codes get english displays
* **language-echo-de-none**: just return codes from an german based code system. All codes get german displays
* **language-echo-en-multi-none**: Just return codes from an english based code system that also has other designations (which are also requested)
* **language-echo-de-multi-none**: Just return codes from a german based code system that also has other designations (which are also requested)

## Specify the language that already exists

Each of the tests below comes as a set of variants, depending on how the language is specified:
* -param: using the displayLanguage parameter
* -vs: using the displayLanguage parameter in the value set
* -header: using the http header
* -vslang: using the value set language
* -mixed: mixing them up (testing precedence)

* **language-echo-en-en**: Specify to return the english code system in english (null operation)
* **language-echo-de-de**: Specify to return the german code system in german  (null operation)
* **language-echo-en-multi-en**: Specify to return the english code system in english, but since the code system has german designations, get them too
* **language-echo-de-multi-de**: Specify to return the german code system in german, but since the code system has english designations, get them too

### Change the language 

Having done these, we turn to the really interesting tests, switching the display language. These tests use the weight feature of the language, 
specifying to return en: ```displayLanguage: en```. By default, the wildcard is understood to be present at some low weight: ```displayLanguage: en, *; q=0.1```. This results in the client getting displays in the language of it's choice, *if there are any*, and otherwise falling back to whatever is available. If the client wants to insist that it only get displays if they are in the specified language, it has to turn the wild card off explicitly: ```displayLanguage: en, *; q=0```

For these tests, it's anticipated that the parameter handling tested out in the previous tests is working, so instead, these tests come in three 
variants:
* -soft
* -default
* -hard

* **language-xform-en-multi-de**: English to German
* **language-xform-de-multi-en**: German to English

### Restricting the designations

Finally, we turn to an efficiency concern - restricting the set of designations that we get back
using the designation parameter. We asked for designations, but we may only want a subset of the 
full set available. We could filter them on the client side, but it might be quite lot of traffic.

* language-echo-en-designation - only spanish designations
* language-echo-en-designations - german and english designations

### Validation

The validation use cases around language include tests for language 
specified as an HTTP header, as a value set parameter, a value set language, 
and for various combinations of language weights.

Note that by default, the wildcard is understood to not be present.
