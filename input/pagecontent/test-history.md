This page details the changes made to the terminology tests over time, based on the GitHub releases. Note that the GitHub repository that contains these tests also contains many other test cases for other kinds of functionality; this history only lists releases that include changes to the terminology tests.

### 1.9.5 (not yet released)

* `$validate-code` with something missing: eighteen new tests in the `validation` suite cover a request with no code at all (a `code` with no `system` and no `inferSystem`, a `system` with no `code`, and nothing at all — all request errors), and a `Coding` or `CodeableConcept` that is missing its `code` or its `system`, against both a value set and a code system. A `Coding` or `CodeableConcept` that was supplied but has bad content is a validation outcome (`result = false`), not a request error, and a missing code is reported against the coding itself, as a warning with issue type `invalid` and a `tx-issue-type` of `invalid-data` (the data supplied was invalid; there is no code for `invalid-code` to be about), with no companion "not in the value set" issue for a code that was never provided
* A `Coding` with neither a system nor a code - a display on its own, say - is **one** problem, not two. The server reports a single warning (`Coding_has_no_system_or_code__cannot_validate`) saying that neither was provided, rather than one message about the missing system and another about the missing code: there is nothing there to validate, and saying so twice does not make it clearer
* A missing `Coding.system` or `Coding.code` is a **warning**, not an error: a code with no system has no defined meaning and cannot be validated, but only the caller knows whether that is acceptable where the coding came from — a validator has no way to tell, and has to report it as a warning either way, so the server reports it as one. The code still does not validate, so `result` is `false`. Two consequences: a `result = false` response does not always carry an error-level issue, and a `CodeableConcept` with a systemless coding alongside a coding that is in the value set still validates (`validation-missing-vs-cc-mixed`). `inferSystem` is defined only for the `code` parameter - it does not apply to a `Coding`, nor to the codings of a `CodeableConcept` - and two tests pin that: with it set, a systemless coding is reported exactly as it is without it

* The `uuid` parameter has been removed from the profiles the tests send (`parameters-default.json`, the two version profiles, and the CDA profile). Nothing ever read it: it was sent on every request, but no server or client uses it, and it is not part of any operation's definition
* `$closure`: a new `closure` suite (mode `closure`) that builds a closure table one concept at a time, checking the entries each addition produces, and then resynchronises. The table is initialised with `reset = true`, which discards anything already held under that name, and the code system it is built over is supplied only in that first request. Suites can now be marked `sequential`, for tests that depend on each other: runners execute them in order, on one thread, after everything else

### 1.9.3

This release is dominated by a rework of the `$translate` tests, and by settling how servers report inactive concepts.

* `$translate`: the source concept may now be named with `sourceCode` + `sourceSystem`, with `sourceCoding`, or with `sourceCodeableConcept`, and R4 servers are tested with the R4 spellings of the same thing (`code` + `system`, `coding`, `codeableConcept`, and `targetsystem`)
* `$translate`: tests for the full range of relationship types, including `not-related-to`, which is returned as a match like any other; and for `ConceptMap.group.element.comment` and `.target.comment`, returned as `sourceComment` and `targetComment` (`element.comment` is preadopted from R6 using a cross-version extension)
* `$translate`: tests for `noMap` — stated as `element.noMap` in R5, and as a target with no code in R4, but reported the same way either way — and for `ConceptMap.group.unmapped` in all three modes (`use-source-code`, `fixed` and `other-map`), including chains of `otherMap` maps and the detection of circular references
* `$translate`: the response reports `originMap`, the concept map the chain of maps started from, and a `used-conceptmap` for every other map that contributed to it (`originMap` was introduced in 1.9.2; it is now settled as the *start* of the chain of maps, not the end)
* `$translate`: the R4 `reverse` parameter is tested — an R4 server reverses the source and target parameters and translates forwards, an R5 or later server returns an error — along with the R5+ way of asking the same question, which is to name the target concept as the source. The translate tests are split into two suites so that the `unmapped` tests see only the concept maps they are about
* Display validation: `validate-code-inactive-display` and `validation-simple-code-bad-display` are each split into a lenient and a not-lenient variant, driven by a new `lenient-display` property on the test case
* Test cases can be restricted to particular FHIR versions with a `version` property (`4.0`, or `!4.0` for "any version but R4"), and `$optional$` accepts a version filter (`version:4`) so that one expected response can cover both R4 and R5 servers where they legitimately differ. Expected content can also be marked `$only$` — required in the nominated versions or modes, and *prohibited* in all the others; this is the counterpart of `$optional$`, which only ever relaxes a requirement
* Inactive concepts: a server that knows a concept is inactive SHALL say why, so the `status` property is no longer optional in an expansion, and the expected responses have been updated to require it. Also, a status of `deprecated` no longer makes a concept inactive — `inactive` and `retired` do (see the [requirements](requirements.html) page)
* A filter that matches no codes: an include whose only filter selects nothing expands to nothing, and must not be treated as an include with no filter, which would select every code in the code system (`simple-expand-regex-none`, `validation-simple-code-regex-none`)
* Repeating concept properties: `CodeSystem.concept.property` is 0..*, so a filter selects a concept when *any* of its values match — not just the first — and `expansion.contains.property` reports all of them (`simple-expand-repeating-prop`, `simple-expand-repeating-prop-values`)
* SNOMED CT: the test subontology has been regenerated from the International August 2025 release with effective time `20250909`, and now includes the MRCM attribute domain and attribute range reference sets, the lateralizable body structure reference set, and a simple reference set. The relationship module in the subset has been corrected, and the loading and regeneration instructions in `tx-source/readme.md` have been rewritten
* SNOMED CT: new postcoordination tests for concept model (MRCM) validation — attribute domain, attribute range, cardinality, grouping and laterality — and for concrete values, both valid and invalid (out of range, a decimal where an integer is required, a concept where a concrete value is required and the reverse); plus an expression whose attribute value is itself refined
* SNOMED CT: a `constraint = *` filter returns every concept, including inactive concepts and module concepts, so the ECL wildcard expansion and `snomed-expand-count-all` now agree (2259 codes)
* Fix the SNOMED CT test set URL: correct the edition/version identifier used across the SNOMED test cases to the terminology-ecosystem test edition (`http://snomed.info/xsct/31000003106/version/20250909`), updating the affected requests and expected responses (display names and version URIs) to match, and correct the type of the `system-version` parameter from `string` to `uri`
* `OperationOutcome.issue.location` is optional in the expected responses, and the `operationoutcome-message-id` extension is optional for every server except tx.fhir.org; the optional `version` OUT parameter is allowed in the six `codeableconcept-*-vs1wb` version tests
* Duplicate canonical URLs resolved: `simple-all` and `simple-enumerated` were each defined twice with different content. The permutations suite now uses the shared `simple/valueset-all.json`, and its own enumerated value set has been renamed to `valueset-simple-enumerated-codes.json`
* Housekeeping: remove test files that were no longer referenced by any test case (the openEHR set, the explicit-version OMOP translate tests, the value set import tests and others), de-duplicate `code-vnn-vsmix-2`, remove a stray `uuid` parameter and stale `$optional$` markers, fix JSON typing errors (a boolean written as a string), add the `total` OUT parameter to the CPT expansion test now that CPT iteration is fixed, and add the missing flat-format expected response for `search-expand-all-yes`

### 1.9.2

This is preparatory for settling the SNOMED CT tests, and then it will be labelled as 2.0.0

* Extensive SNOMED CT ECL testing: reorganised and greatly expanded the ECL tests — operator grouping and precedence (grouped-or / grouped-and, ambiguous precedence), cardinality and role-group occurrence counting, complex refinements and exclusions, and wildcard expansions; require `valueset-unclosed` on filter-based ECL expansions; and tests for enumerating grammar-based code systems (including `count=0` counts)
* Add test cases for the new `$compare` operation (previously named `$related`)
* Translate improvements: reverse translation, and `originMap` (replacing `sourceMap`)
* More code system version-control tests: value sets including/excluding different versions of the same code system, version pinning across all SNOMED requests, and version-dependent translate tests
* Tests for complex exclusions, contained value sets, `child-of`, and regex filters — including a regex denial-of-service case and improved regex error messages
* Search `$expand`: flat result format (`response:flat`) and support for the `total` OUT parameter
* Warnings for status / retired / inactive codes, with fixes to the reported location
* Display-name consistency, additional language and secondary-display tests, and allow `designation.use.display` in responses
* Validate a `CodeableConcept` using the first matching code rather than the last
* Content refresh: LOINC 2.82, updated SNOMED test load-set and version URIs, and updated VSAC content
* Numerous renames, parameter-name and typo fixes for consistency, unique value set ids, and removal of stale parameters (non-standard `limit` replaced by `count`, bad `cache-id`)

### 1.9.0

* Add a set of tests for controlling the version of CodeSystems (`ValueSet.compose.include.version`, with wildcards, and the expansion parameters `system-version` etc.)
* Revisit the tests to ensure that they are correct
* Remove the deprecated `version` parameter that is no longer used or supported (after 1 year of grace)
* Various improvements to error messages and test descriptions

### 1.7.7-SNAPSHOT

* Define `hierarchyMeaning` for the simple CodeSystem

### 1.7.6

* Allow `expansion.id` & `expansion.offset` in many tests
* Rename `valueset-version` to `default-valueset-version`
* tx.fhir.org only: Add supplement test cases and LOINC tests for CLASSTYPE, answers-for, and answer-list

### 1.7.5

* Add test for validation of `displayLanguage`

### 1.7.4

* Add message id

### 1.7.3

* Use `systemVersion` instead of `version`

### 1.7.2

* Add message-id extension to tests

### 1.7.1

* Remove language weights except where it matters
* Add `x-caused-by-unknown-system` when a supplement is not found
* Remove R4 variants

### 1.7.0

* Move master tests to tx-ecosystem-ig instead of general test cases

### 1.6.6

* Fix references to wrong value set in supplement tests

### 1.6.2

* Add specific tests for the interaction between `ValueSet.compose.inactive` and the `activeOnly` parameter
* Add tests for correctly populated CapabilityStatement and TerminologyCapabilities resources

### 1.6.0

There is no specific history for the terminology test cases prior to version 1.6.0. The only history notes available are mixed in with all the other kinds of tests in the GitHub release notes.
