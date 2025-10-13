### Requirements

This document describes the requirements for all servers that are part of the [HL7 terminology ecosystem](ecosystem.html).
Note that systems do not need to conform to these requirements to be described as 'FHIR Terminology servers',
but they do not need to conform to these requirements to be part of the ecosystem.

All systems need to conform to the following requirements:

### General Requirements 


### Capabilities 

* The server SHALL return a CapabilitiesStatement from /metadata


### Terminology Capabilities 


### Expand 


### Code system validation 


### Value set validation 



#### Metadata

* The server SHALL return a CapabilityStatement from ```{root}/metadata```
* It SHALL populate the `CapabilityStatement.fhirVersion` and `CapabilityStatement.rest[mode = server].security.service properties`
* It SHALL include a `CapabilityStatement.instantiates` value of `http://hl7.org/fhir/CapabilityStatement/terminology-server`
* The server SHALL return a TerminologyCapabilities statement from `{root}/metadata?mode=terminology`

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

### Passing CodeSystem resources in requests 

Terminology servers can choose to accept CodeSystems in the [tx-resource parameter](https://jira.hl7.org/browse/FHIR-33944). 
General purpose servers SHALL do this (and are required to do this to pass the general tests).

* Servers SHALL indicate their support or not for passing CodeSystem resources in the tx-resource parameter using [tbd]

Terminology servers SHOULD support passing CodeSystem supplements, particularly language packs. Servers that don't support 
language packs should only choose not to support language packs when there is governance over the use of language translations,
and only by negotiation with the terminology ecosystem managers (see also notes below)

* Servers SHALL indicate their support or not for passing CodeSystem supplements in the tx-resource parameter using [tbd]

* Servers intended to be used as the primary server for the validator and IG publisher SHALL accept CodeSystems in the 
  [tx-resource parameter](https://jira.hl7.org/browse/FHIR-33944). (e.g. this rule does not apply servers registered with the ecosystem)


##### Code system Functionality

Servers are required to support code system supplements. Specifically, this means:

* Servers SHALL not ignore supplements, though they MAY choose to return errors rather than process them correctly

Servers are required to support the following properties in the CodeSystem resource:

* `CodeSystem.caseSensitive` SHALL be supported: validation SHALL correctly check case. In the case of non-case sensitive code systems, expansions SHOULD just contain the code as defined (in the code system or the value set enumeration), and not all the case variants that could be generated.

* `CodeSystem.valueSet`. Tbd what this means

* `CodeSystem.hierarchyMeaning`. If the code system defines a hierarchy, both `$expand` and `$validate` SHALL correctly handle the hierarchy when interpreting filters and validating codes 

* `CodeSystem.compositional`. A server SHALL not accept compositional grammars for codes unless the CodeSystem is marked as Compositional. Servers that do not know the grammar for a CodeSystem that is marked as compositional SHALL note this in validation errors for the CodeSystem

* `CodeSystem.versionNeeded`. If a CodeSystem says that version is needed, the `$validate-code` operation SHALL check that version is populated, and return an error if it's not

* `CodeSystem.content`. Servers SHALL not process `$expand` or `$validate-code` requests on CodeSystems that have `content = not-present` or `example`. Servers SHALL reflect `content = fragment` in an error message if the code is not valid against a fragment. 

* `CodeSystem.supplements`. Servers SHALL not mistake supplements and code systems for each other.

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

#### Human Representation 

* All pre-defined code systems and value sets SHALL have a web representation (as above) 

The content on that page MAY be static or active; it is at the discretion of the server to decide what's on the page, but it SHOULD be more than just the json/xml for the resource (and it isn't limited to information in the resource, e.g. the server MAY choose to make additional process/provenance/context information available)

The server MAY choose to make this content available at the end-point for the relevant resource. e.g. a request for `{root}/CodeSystem/123` with an ```Accept``` header of 'application/fhir+json' returns the resource, and the same URL with an ```Accept``` header of 'text/html' returns a web page suitable for human consumption. Servers are not required to do this; they MAY choose to make the content available elsewhere.

If the server chooses to make them available elsewhere, it SHALL populate the extension ```http://hl7.org/fhir/StructureDefinition/web-source``` in any resources it makes available with a `valueUrl` where the web view can be found. This SHALL be populated when the CodeSystem and ValueSet are read, and also in any `$expand` of the value set (just for the root value set in this case).

#### Parameter Support

##### Common Parameters ($expand and $validate-code)

* The server SHALL support the [tx-resource](https://jira.hl7.org/browse/FHIR-33944) parameter for passing related terminology, per the rules above
* If a server receives a terminology resource it does not process correctly, it SHALL return an error

* The server SHOULD support the [cache-id](https://jira.hl7.org/browse/FHIR-33946) parameter
* If it does, it SHALL report so in `TerminologyCapabilities.expansion.parameter.name` (this parameter makes a big difference to performance for clients, significantly reducing network utilization)

* The server SHALL support supplements for the purposes of designations in different languages
  * Clarification: Servers SHALL not ignore supplements; if they don't support a relevant supplement (per the rules above), they SHALL return an error that they cannot process the supplement (e.g. if it is passed in a tx-resource parameter)
  * Whether servers actually accept and use supplements for the purposes of designations is a matter for negotiation with the server's relevant user base and whether there are other arrangements in place for supporting translation (e.g. SNOMED CT, LOINC)

##### $expand parameters

* The server SHALL support the `count` parameter (`offset` is used, but will always be 0)
* The server SHOULD return hierarchical expansions when possible (this is not a technical requirement, but comes up as important to authors)
* The server SHALL support `system-version`, `check-system-version`, and `force-system-version`
* The server SHALL echo all parameters (including assumed values) in the expansion parameters
* The server SHALL report with all versions of code systems used (in 'used-*')
* The server SHALL support language correctly (both `displayLanguage` parameter and Accept-Language header, as specified in [Languages](./languages.html))
* The server SHALL support the `excludeNested`, `includeDesignations`, `activeOnly`, `includeDefinition`, `property`, `designation` parameters

##### $validate-code parameters

* The server SHALL support validating code+system(+version)(+display), `Coding`, and `CodeableConcept`
* The server SHALL support the [mode/valueSetMode](https://jira.hl7.org/browse/FHIR-41229) parameter
* The server SHALL support language correctly (same locations/rules for `$expand`)
* The server SHALL support the [inferSystem](https://jira.hl7.org/browse/FHIR-41431) parameter
* The server SHALL return an `issues` parameter when there are issues to return 
* The server SHALL return system, code and display for the code that it considered valid, along with the version, if this is known
* The server SHOULD return a `x-caused-by-unknown-system` parameter for each code system it did not support
* The server SHOULD return a `normalized-code` parameter where appropriate (e.g. case insensitive code systems, code systems with complex grammars)
* The server SHOULD return an issue with tx issue type ```processing-note``` when it has not fully validated the code e.g. an SCT expression against the MRCM 

##### Inactive Codes 

Code systems vary in whether codes and/or designations can be labelled as inactive, and if they do, how it is done. 
SNOMED CT defines 'inactive' explicitly. For other Code Systems, codes or designations are labelled as 'should not use' 
in any fashion, they are regard as inactive.

For the CodeSystem resource:
- a concept that as a 'http://hl7.org/fhir/concept-properties#status' property value of 'deprecated' or 'retired' is inactive
- if there is no status property for a concept, the standards-status extension (see below) may provide the status
- for a designation, the only way to denote inactive (at this time) is to use the standards-status extension.

If a concept is defined as inactive:
* inactive SHALL be true when the concept is found in an expansion 
* in expansions, the status property SHALL be populated with a status indicating why the concept is inactive (usually 'inactive', 'withdrawn', or 'deprecated')
* the return value from $validate-code for the concept SHALL include a warning that the code is invalid, and the parameters 'inactive' and 'status' SHALL be populated

If a designation is defined as inactive:
* if the designation is included in an expansion, it SHALL either have a standards-status extension with a value of either 'withdrawn' or 'deprecated', or a use code of 'http://snomed.org/info#900000000000546006'
* if provided as the display to $validate-code, an inactive display causes a warning that the display is no longer current (but it is valid)


##### Extensions

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
