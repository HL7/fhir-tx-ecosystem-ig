This page details the changes made to the terminology tests over time, based on the GitHub releases. Note that the GitHub repository that contains these tests also contains many other test cases for other kinds of functionality; this history only lists releases that include changes to the terminology tests.

### 1.10.0

This is a big release that covers a lot of decision making and development associated with the cluster of terminology meetings in Sydney in October 2026. 

#### Testing Infrastructure 

* Features: a suite or test can now depend on what the server declares in its CapabilityStatement, with `feature` - the operations it implements (`expand`, `validate`, `lookup`, `subsumes`, `translate`, `closure`, `related`), and `tx-resource` for the `CodeSystemAsParameter` feature. See [Features](testcases.html#features). This needs a test runner that knows about features
* Messages: about 440 expected responses now give message text as `$external`, so that a server can provide its own wording in an externals file instead of matching tx.fhir.org's (see [Test Cases](testcases.html)). An externals file for `tx.health-samurai.io` has been added

#### New Functionality 

* `$closure`: a new `closure` suite (11 tests) builds a closure table one concept at a time - initialising it, adding codes, resynchronising from a version, and disposing of it (`closure-dispose`). The tests depend on each other, so the suite runs in order, on one thread, after the other suites. It needs the `closure` feature: a server runs it when it declares `$closure`, whatever modes the tests are run with. R4 servers can pass it, with the R4 equivalence codes

#### Corrections

* `$lookup`: `name` may be either the CodeSystem `name` or its `title`. When a code is not selectable, `$lookup` always returns both `abstract` (`true`) and the `notSelectable` property (`true`), whether or not they were asked for; when it is selectable, it returns neither unless asked. A `displayLanguage` that the code system has no content in (`zxx`) gets the default display and a warning (`simple-lookup-zxx`, `lookup-mms-zxx`)
* Contained value sets: a `#id` reference is resolved against the `contained` resources of the containing resource, so a contained value set can import another value set contained in the same resource (`simple-expand-contained-nested`, `validation-contained-nested`); contained value sets that import each other are a circular reference (`simple-expand-contained-circular`). The requirements now say explicitly that value sets contained in value sets are the only use of contained resources servers are required to support
* Excluding a whole value set - an exclude with only a `valueSet` - had no test of its own (`exclude-combo` excludes a value set together with a code system and concepts); `exclude-valueset` excludes one by url, and `exclude-contained` a contained one by `#id`. An excluded value set is reported as a `used-valueset`, as an imported one is
* `$translate`: the server checks each target concept it returns against its code system. A target that is not valid is still returned as the map states it, with an `error` parameter saying that the target code is not valid (`translate-2b`, `translate-5`, `translate-5b`, `translate-ciel4`); a target from a code system the server does not know carries a `note`. A target that is not in its code system's normal form also gets a `normalized-concept`, and a source code is matched both as given and in its normal form; the map's own codes are not normalised
* Global language packs: the `codesystem-supplement-type` code `lang-pack` is renamed `global-lang-pack`, and servers have to support global language packs. A new `global-lang-pack` suite (mode `general`) checks that a global language pack passed in `tx-resource` is used for the code system it supplements without being referenced - in `$validate-code`, `$lookup` and `$expand`, where it is reported as a `used-supplement` - and only for that code system; and that any other supplement passed in `tx-resource` is only used once it is referenced. Until now this was only tested in the `tx.fhir.org` mode
* Unknown properties: a property the server does not know, asked for with the `property` parameter (`x-unknown-property`), is ignored - not an error or a warning - by `$lookup` (`simple-lookup-unknown-property`, `simple-lookup-only-unknown-property`) and `$expand` (`simple-expand-unknown-property`, where it is not declared in `expansion.property`). The first test also checks that a property asked for by its own code is returned
* Inactive codes: in a `CodeableConcept`, the inactive warning names the inactive coding, not the first one, and a later active coding does not suppress it (`inactive-4-validate`, `inactive-5-validate`)
* Requirements: `$validate-code` supports `valueset-membership-only` (this replaces the `mode`/`valueSetMode` requirement); a value set that imports itself, directly or through other value sets, is an error, but one that is imported more than once by different routes is not
* Smaller changes: a flat-mode expected response for `vs-expand-versionless`, a fix to the R4 expected response for `omop-translate-loinc-implicit`, and corrected messages in `validation-simple-code-implied-bad-code`. In five of the 1.9.5 missing-code tests (`validation-missing-*`), `$optional$` was the string `"true"`, which is read as a mode name and so made the content required for every server: the `operationoutcome-message-id` extension is now optional for every server except tx.fhir.org, as elsewhere, and the echoed `code` in `validation-missing-cs-coding-no-system` is optional

#### Terminology Specific Testing

##### ICD-11 (`icd-11` suite)

82 new tests, and the code form has changed: in the MMS and ICF the code is the bare entity id, and the full entity URI is no longer a code (`lookup-mms-uri` and `cs-validate-uri` now expect it not to be found; `lookup-mms-id` and `cs-validate-id` use the id). New tests cover:

  * the `classificationCode` extension on a Coding, in `$validate-code` against the code system and value sets (`cs-validate-coding-cc-*`, `vs-validate-coding-cc-*`)
  * the normal form of a postcoordinated expression (`cs-validate-nf-*`): qualifiers in the order of the root's axes, several values on one axis sorted by entity id, cluster members ordered by axis, duplicates removed, and `normalized-code` in the id form
  * `$lookup` of postcoordinated expressions (clusters, injuries, retinopathy, pneumonia), with and without metadata, and of chapters, windows, residual categories, scales, sources and languages
  * `$expand` totals for MMS, ICF, the Foundation, the Extensions and the Medicaments, ad hoc value sets in the id form, and `$subsumes`
  * `$translate` with ConceptMaps to and from CIEL that use codes and `classificationCode` expressions (`translate-ciel*`)
  * `abstract` is required in `$lookup` of a concept that is not selectable, and the expansion timestamp is optional

##### SNOMED CT (`snomed` suite)

* the test edition now includes a fragment of the LOINC Extension (module `LO1010000`), and new tests cover it: `$lookup` returns the LOINC code as a property, by default too, and a concept with no LOINC code has none (`snomed-lookup-loinc*`); `$expand` of LOINC Extension concepts, with and without a LOINC code (`snomed-expand-loinc*`); and `$translate` between SNOMED CT and LOINC in both directions (`snomed-translate-loinc*`)
* translating from a national edition to the International Edition, with no ConceptMap (`snomed-translate-edition-*`), using extension concepts added to the test edition (module `TEST1000003`): an extension concept equivalent to an International concept, a defined concept that becomes a post-coordinated expression (grouped and ungrouped), a primitive concept that translates to its nearest International ancestor, an inactive concept with and without a replacement, and concepts with no useful translation (directly under a top-level hierarchy, or with no International ancestor at all). Where a post-coordinated expression is produced, it uses only International Edition concepts
* the concepts added to the test edition change the counts in `snomed-expand-count-all` and the ECL wildcard expansions, so servers need to reload the test edition
* ECL (`sct-ecl` suite): 34 new tests - member filters (including dates, and a misplaced filter), top and bottom of set, operators applied to sets and to `memberOf`, `or` and `and` in refinements and inside groups, attribute names given as expressions, sets and wildcards, dotted attributes, comments, concept filters (`definitionStatus`, `effectiveTime`, `active`), description filters (term, term sets and wildcards, `typeId`, `language`, `active` and its default, and two filter blocks), and `memberOf` with descriptions and field selection
* an attribute the MRCM marks as grouped, but written outside a relationship group (the Close To User Form of the SNOMED postcoordination guide), is no longer an error. The server applies the "Adding a Self-grouped Attribute" transformation, so `40468003:363698007=10200004` is equivalent to `40468003:{363698007=10200004}` for validation, rendering and subsumption (`subsumes-expr-grouped-ungrouped` and `subsumes-expr-ungrouped-morphology` now expect a result, not an error). The [requirements](requirements.html) page also records the scope of post-coordination support agreed by the SNOMED Computable Languages Group; the parts that are not yet tested are marked

##### ICF (Pre ICD-11) (mode `icf`)

A new suite and mode `icf` that tests the ICF qualifier grammar - `.` then one qualifier digit per position, where the positions depend on the component: extent for body functions (`b`); extent, nature and localization for body structures (`s`); performance, capacity, and optionally capacity with assistance and performance without assistance, for activities and participation (`d`); and one for environmental factors (`e`), a barrier after `.` or a facilitator after `+` in place of the `.` (no `+9`). 8 (not specified) fills a position and is left out of the generated display.

* `$validate-code` of the  
* The tests cover bad qualifiers, too many qualifiers, headings and the generated displays
* `$expand` of an enumerated value set of qualified codes
* and value sets that control qualifiers with filters: each qualifier `exists` `true`/`false` (a qualifier of 8 does not count as present) and `qualified` `=` `false` (no qualified codes at all)

##### LOINC (`loinc` suite)

* The LOINC tests have moved out of the `tx.fhir.org` suite into their own `loinc` suite and mode, and their files from `tests/tx.fhir.org` to `tests/loinc`
* Property values and filters (see [Requirements](requirements.html)): the six axes take the Part code, and `CLASS` the class name. `$lookup` now returns `CLASS` as a string (`CHEM`), optionally with the Part code as `partCode`. New tests filter `SCALE_TYP` and `METHOD_TYP` by Part code (`loinc-expand-scale-type-part`, `loinc-validate-filter-scale-type-part-*`, `loinc-expand-prop-method-part`), and `CLASS` = `PULM` with `SCALE_TYP` = `LP7753-9` (`loinc-expand-class-scale`, `loinc-validate-filter-class-scale-*`, from the MII lung function value set). The legacy forms - an axis filtered by name (`DOC`, `Glucose`), `CLASS` filtered by Part code (`loinc-expand-class-part-scale`, `loinc-validate-filter-class-part-scale-good`) - stay, and all servers are required to support them. `loinc-validate-filter-scale-type-bad` used the wrong value set, and `loinc-expand-prop-method` is now documented as deliberately filtering on the wrong axis
* Updated to 2.83, and the expected responses updated to match. Every request now names version 2.83: `version` in `$lookup`, `$subsumes` and `$validate-code` on the code system, `systemVersion` in `$validate-code` on a value set, `system-version` in `$expand` (echoed in `expansion.parameter`), and the version of the source Coding in `$translate` (echoed in `sourceConcept`)
* `is-a` includes the concept itself, as FHIR defines it, so `loinc-expand-concept-is-a` now includes `LP14082-9` (431 codes, not 430). The LOINC hierarchy is read from `ComponentHierarchyBySystem` path by path: a code is a descendant of the parts on its own `PATH_TO_ROOT`, and the children a part has under one parent are not inherited by its other parents
* a part's mappings to other code systems (`PartRelatedCodeMapping`) are Coding properties named for the map - `sct`, `ncbi`, `chebi` - with the equivalence as a subproperty, and are returned by default (`loinc-lookup-part-mappings`); an answer's external code from the CDC answer list is reported in the CDC race and ethnicity system as the property `race` (`loinc-lookup-answer-race`)
* a filter for terms with no method - `METHOD_TYP` `exists` `false` - in `$expand` and `$validate-code` (`loinc-expand-filter-no-method`, `loinc-validate-filter-no-method-*`)
* `$translate` between LOINC and other code systems using the maps that LOINC publishes: parts to and from SNOMED CT (`loinc-translate-part-sct`, `-sct-url`, `loinc-translate-sct-part`), and codes to ISO/IEEE 11073 MDC (`loinc-translate-code-mdc`, where the MDC codes are narrower than the LOINC code)

### 1.9.6

This release mostly corrects expected responses that were wrong, but were not being checked properly.

* Test runner: when two strings did not match, the runner fell back to comparing them as base64, and the base64 decoder skips any character outside the base64 alphabet and stops at the first `=`. So any string containing an `=` - most SNOMED CT expression messages, and anything naming an implicit value set (`?fhir_vs=`) - was only compared up to its first `=`. The runner now only compares strings as base64 when they really are base64 (this is fixed in the test runner, not the test cases; runners up to 6.10.5 have the bug). Correcting it exposed the wrong expected responses listed below
* SNOMED CT: an unknown code message names the edition, and the test edition is the `Test Edition`, not the `International Edition` (`validate-code-procedure-pc-bad1` and `-bad2`, `validate-code-pc-unbracketed`, and the `validate-code-pc-mrcm-*` tests)
* SNOMED CT: `validate-code-implied-2` named the wrong value set in its message - it is the implicit value set that was asked for (`http://snomed.info/sct?fhir_vs=refset/734139008`), not `sct-pc-none`; and the display of 80891009 in `bugs/validate-sct-parse-pc` is `Heart structure`
* Smaller message corrections that had been hidden the same way: the display-comment messages in the inactive display tests (`validate-code-inactive-display-*`, both SNOMED CT and `extensions`), the list of matching systems in `errors-combination-bad`, the quoted code system URL in the unknown code system messages (`errors-unknown-system2`, `validation-simple-coding-bad-system`), and a stray double space in the `related` `simple-all` message
* Other expected content corrected: the ECL expansion responses had a stray `}` at the end of the value set URL; the overload tests expect the display `Display #2`; the OMOP `used-conceptmap` has no trailing `|`; and the LOINC supplement in the `tx.fhir.org` `$lookup` tests is version 2.82
* ValueSet `$validate-code`: the code system version is `systemVersion`, not `version` (which is the `CodeSystem/$validate-code` parameter, and is not defined for `ValueSet/$validate-code`). Seven SNOMED CT and ICD-11 requests used `version`, which servers were right to ignore; they now use `systemVersion`, and `validate-code-pc-none` now states it too
* `validate-code-pc-none`: when the code is not in the value set, the version reported is the one the value set selects (`compose.include.version`, here the test edition), not the server's default edition
* `$translate`: `result` can only be `true` if at least one match is an actual translation. A match that says the concept is not related (`not-related-to`, or `disjoint` / `unmatched` in R4), or that there is no mapping (`noMap`), is still returned, but `result` is `false` (`translate-2b`, `translate-4`)
* Messages that legitimately vary between servers are now checked for their content rather than their wording: the expression error in `validate-code-procedure-pc-bad1` and `validate-code-pc-unbracketed` must mention `expression` (and the code at fault), and the not-in-value-set message in `validate-code-pc-none` must name the value set and the expression
* `displayLanguage` is written without spaces (`en,*;q=0`, not `en, *; q=0`), in the requests and in the expansion parameter a server echoes back (`language-xform-*-hard`, `validation-simple-coding-bad-language`). The `Accept-Language` header in `validation-simple-coding-bad-language-header` keeps its spaces, which are legal in a header

### 1.9.5

* `$validate-code` with something missing: eighteen new tests in the `validation` suite cover a request with no code at all (a `code` with no `system` and no `inferSystem`, a `system` with no `code`, and nothing at all — all request errors), and a `Coding` or `CodeableConcept` that is missing its `code` or its `system`, against both a value set and a code system. A `Coding` or `CodeableConcept` that was supplied but has bad content is a validation outcome (`result = false`), not a request error, and a missing code is reported against the coding itself, as a warning with issue type `invalid` and a `tx-issue-type` of `invalid-data` (the data supplied was invalid; there is no code for `invalid-code` to be about), with no companion "not in the value set" issue for a code that was never provided
* A `Coding` with neither a system nor a code - a display on its own, say - is **one** problem, not two. The server reports a single warning (`Coding_has_no_system_or_code__cannot_validate`) saying that neither was provided, rather than one message about the missing system and another about the missing code: there is nothing there to validate, and saying so twice does not make it clearer
* A missing `Coding.system` or `Coding.code` is a **warning**, not an error: a code with no system has no defined meaning and cannot be validated, but only the caller knows whether that is acceptable where the coding came from — a validator has no way to tell, and has to report it as a warning either way, so the server reports it as one. The code still does not validate, so `result` is `false`. Two consequences: a `result = false` response does not always carry an error-level issue, and a `CodeableConcept` with a systemless coding alongside a coding that is in the value set still validates (`validation-missing-vs-cc-mixed`). `inferSystem` is defined only for the `code` parameter - it does not apply to a `Coding`, nor to the codings of a `CodeableConcept` - and two tests pin that: with it set, a systemless coding is reported exactly as it is without it
* The `uuid` parameter has been removed from the profiles the tests send (`parameters-default.json`, the two version profiles, and the CDA profile). Nothing ever read it: it was sent on every request, but no server or client uses it, and it is not part of any operation's definition
* `$subsumes` with no code system (`simple-subsumes-no-system`, `simple-subsumes-no-system-coding`): `codeA` and `codeB` with no `system` parameter, or a `codingA` and `codingB` with no system, is a problem with the request, not a code system that cannot be found - `400`, with issue type `invalid`, a `tx-issue-type` of `invalid-data`, and the message id `SUBSUMPTION_NO_SYSTEM`
* SNOMED CT: an attribute value that is itself refined must be bracketed. The compositional grammar allows only a bare concept reference as an attribute value; anything more is a bracketed sub-expression, so the correct form is `367430006:{405813007=(85562004:272741003=24028007)}`. Without the brackets the expression is ambiguous as well as invalid - in `A:{B=C:D=E,F=G}` nothing says whether `F=G` refines C or A. `lookup-pc`, `validate-code-pc-nested-good`, `validate-code-implied-2` and `validate-code-implied-2b` used the unbracketed form, and have been corrected (including the expected displays); two new tests check that the unbracketed form fails (`lookup-pc-unbracketed`, `validate-code-pc-unbracketed`)
* SNOMED CT: the comma before an attribute group is optional in the compositional grammar, so an attribute set followed directly by a group has to be accepted, and normalises to the form with the comma (`validate-code-pc-scg-no-comma`, `validate-code-pc-scg-comma`)
* SNOMED CT: an attribute with more than one domain in the concept model is valid in any of them, and not outside all of them (`validate-code-pc-mrcm-domain-finding`, `-event` and `-multi`)
* Mime types: subsumption where a parameter has a default value. Bare `text/plain` means `charset=us-ascii` (RFC 6657) and `format=fixed` (RFC 3676), so writing the default out gives the same media type, and a different value contradicts the bare form rather than narrowing it - this is where the rule that a parameter narrows a media type does not hold. For a `text/*` type whose registration does not say how its charset is determined, a server cannot tell whether a missing charset means any charset or US-ASCII, so the relationship cannot be determined; and an unknown parameter carried identically by both codes does not stop the server deciding
* `$translate`: `ConceptMap.sourceScope` limits the scope of a map, so a code outside it has no applicable map, and the map's `unmapped` does not apply either (`translate-6a`, which has been corrected); naming the map explicitly does not put the code into its scope (`translate-6b`)
* Draft content: a reference to a draft code system from an active value set is now reported as `MSG_DRAFT_SRC_STATUS`, and the message names the value set as well as the code system
* Smaller changes to expected responses: the SNOMED CT `$lookup` `name` is the edition name (`SNOMED CT Test Edition`), not `system|version`; `validate-sct-display-2` expects the US English display now that refset language handling is fixed; the ICD-11 `cs-validate-uri` issue is `CODE_NOT_IN_NORMAL_FORM` rather than `CODE_CASE_DIFFERENCE`; the capability statement must declare `$subsumes`; and `bugs/validate-no-system` and `validation-simple-coding-no-system` follow the missing-system rules above

### 1.9.4

This release is mostly about `$subsumes`, which previously had almost no tests, and adds suites for mime types, ICD-11 and `ValueSet.compose.property`.

* `$subsumes` against a `CodeSystem` resource: the `simple-subsumes-*` tests cover parent, child, equivalent, transitive, siblings, unrelated roots and an unknown code, each with both `codeA`/`codeB` and `codingA`/`codingB`. The `tho` suite does the same for act-class, where the hierarchy is stated in `subsumedBy` properties rather than by nesting, including a code with two parents - a relationship nesting cannot express at all - and two parents of the same code, which are not thereby related to each other
* Filters on a hierarchy stated by properties rather than nesting: `is-a`, `descendent-of`, `is-not-a` and `child-of` against act-class, and `child-of` in the simple code system (`validation-simple-child-of-child`, `-grandchild`). `descendent-of` excludes the code named in the filter, which is what distinguishes it from `is-a`; `child-of` excludes it too, and stops at the direct children, which is what distinguishes it from `descendent-of`
* SNOMED CT `$subsumes`: simple codes (including an unknown code, a malformed code, an unknown edition and an unknown code system), and 17 tests for post-coordinated expressions - refined expressions against ancestors and their own focus concept, attribute values, extra and different attributes, a primitive focus concept, a concept against its own normal form, grouping as the MRCM requires it, terms, conjunctions, and expressions against precoordinated concepts
* `$subsumes` can fail to decide. It has no outcome for "unknown", so where a server cannot determine the relationship it returns an error with the `tx-issue-type` `cannot-determine` rather than claiming `not-subsumed`. This applies to SNOMED CT expressions where the structure finds no relationship (which is not a proof that none exists - a server that can classify may answer `not-subsumed` instead), and to mime types with a parameter the server does not know
* `$subsumes` for LOINC (in the `tx.fhir.org` suite) and UCUM. UCUM has no hierarchy: the same unit written two ways is equivalent, but a scaled unit, or a unit with the same canonical unit and a different magnitude, is not subsumed
* Language codes (BCP 47): validation against the subtag registry (private-use region ranges, grandfathered tags, and the `Prefix` rules for extlang and variant subtags); case - a tag in the wrong case is valid, since BCP 47 is case-insensitive, but the server returns `normalized-code`; subsumption, following RFC 4647 extended filtering; `language`, `region` and `script` filters, where an absent component does not match a fixed one; and expansion of those filters, which is paged for a language, too costly for a region, and not possible for a script alone
* Mime types: a new `mimetypes` suite (mode `mimetypes`). Type and subtype have no hierarchy, but parameters narrow a media type, and a structured suffix is tested too. The `base` filter selects by type or type/subtype and the `registered` filter by presence in the IANA registry; only `registered = true` can be expanded, and that expansion is marked unclosed
* ICD-11: the `icd-11` suite (mode `icd-11`) now has tests for `$lookup` (MMS short codes and entity URIs, groupers, residual categories, the Foundation, ICF, languages and postcoordination), `$validate-code` against the code system and against value sets, and `$expand` of the WHO postcoordination scale value sets and of client-supplied value sets. Several are expected to fail against the current WHO ICD-API; `tests/icd-11/doco.txt` says what each one asserts and why
* `ValueSet.compose.property`: a new `properties` suite that expands value sets asking for all properties, with a wildcard and by enumerating them
* `validation-simple-codeableconcept-unknown-system`: a `CodeableConcept` with a coding from an unknown code system alongside a coding that is in the value set. The unknown system cannot be checked, but the other coding satisfies membership, so the result is `true`, with a warning
* Inactive codes are always reported as inactive, in the LOINC and SNOMED CT `$validate-code` tests in the `tx.fhir.org` suite
* `OperationOutcome`: `diagnostics` is no longer asserted in any expected response - it is for server-specific detail - and every issue now carries a `tx-issue-type` coding (`too-costly` and `not-supported` were the ones missing)
* `ConceptMap-novs` has been given its own id: it had the same id as `ConceptMap-full`

### 1.9.3

This release is dominated by a rework of the `$translate` tests, and by settling how servers report inactive concepts.

* `$translate`: the source concept may now be named with `sourceCode` + `sourceSystem`, with `sourceCoding`, or with `sourceCodeableConcept`, and R4 servers are tested with the R4 spellings of the same thing (`code` + `system`, `coding`, `codeableConcept`, and `targetsystem`)
* `$translate`: tests for the full range of relationship types, including `not-related-to`, which is returned as a match like any other; and for `ConceptMap.group.element.comment` and `.target.comment`, returned as `sourceComment` and `targetComment` (`element.comment` is preadopted from R6 using a cross-version extension)
* `$translate`: tests for `noMap` — stated as `element.noMap` in R5, and as a target with no code in R4, but reported the same way either way — and for `ConceptMap.group.unmapped` in all three modes (`use-source-code`, `fixed` and `other-map`), including chains of `otherMap` maps and the detection of circular references
* `$translate`: the response reports `originMap`, the concept map the chain of maps started from, and a `used-conceptmap` for every other map that contributed to it (`originMap` was introduced in 1.9.2; it is now settled as the *start* of the chain of maps, not the end)
* `$translate`: the R4 `reverse` parameter is tested — an R4 server reverses the source and target parameters and translates forwards, an R5 or later server returns an error — along with the R5+ way of asking the same question, which is to name the target concept as the source. The translate tests are split into two suites so that the `unmapped` tests see only the concept maps they are about
* Display validation: `validate-code-inactive-display` and `validation-simple-code-bad-display` are each split into a lenient and a not-lenient variant, driven by a new `lenient-display` property on the test case
* Test cases can be restricted to particular FHIR versions with a `version` property (`4.0`, or `!4.0` for "any version but R4"), and `$optional$` accepts a version filter (`version:4`) so that one expected response can cover both R4 and R5 servers where they legitimately differ. Expected content can also be marked `$only$` — required in the nominated versions or modes, and *prohibited* in all the others; this is the counterpart of `$optional$`, which only ever relaxes a requirement
* Inactive concepts: a server that knows a concept is inactive must say why, so the `status` property is no longer optional in an expansion, and the expected responses have been updated to require it. Also, a status of `deprecated` no longer makes a concept inactive — `inactive` and `retired` do (see the [requirements](requirements.html) page)
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
