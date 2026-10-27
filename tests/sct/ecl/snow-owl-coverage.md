# Snow Owl ECL test cases marked "Add to IG" - coverage in the tx-ecosystem suite

Source: the Snow Owl ECL test spreadsheet (Sheet2), rows with Add to IG = 1, as marked by the chair of the SNOMED committee. 
Status: `existing` - the shape was already in the suite (including tests added October 2026); `new` - added from this list, adapted to the SNOMED test edition; `consult` - could not be adapted, see the note. All expected results are from the test edition at version 20250909 (`http://snomed.info/xsct/31000003106/version/20250909`), including the LO1010000 and TEST1000003 modules.

| # | Snow Owl test | ECL (Snow Owl) | Status | IG test(s) | Note |
|---|---|---|---|---|---|
| 0 | syntaxError | `invalid` | existing | snomed-expand-ecl-invalid-sctid, -missing-focus, -trailing-tokens |  |
| 1 | _null | `(null)` | consult |  | A null / empty constraint value: Snow Owl rejects; the FHIR equivalent is a filter with an empty value. Should the IG expect an error (the ECL grammar requires an expression) or an empty expansion? |
| 2 | whitespaces | `(whitespace only: space, newline, space, tab)` | consult |  | Whitespace-only ECL: Snow Owl returns an empty result; FHIRsmith reports a syntax error. The grammar requires an expressionConstraint, so an error seems right - to confirm. |
| 3 | memberOfAny | `^*` | new | snomed-expand-ecl-memberOf-any | Description-member reference sets (language refsets) contribute no concepts. |
| 4 | memberOfNested | `^(< 900000000000538005 \|Description format reference set\|)` | new | snomed-expand-ecl-memberOf-nested | Adapted to ^ (< 900000000000455006 |Reference set|): the Description format refset is not in the test edition. |
| 5 | descendantOfAny | `<*` | new | snomed-expand-ecl-descendant-of-any | count=0 (total only). The test edition has 11 roots, not 1: the subset extraction dropped the is-a relationships of module/metadata concepts, and inactive concepts have none. |
| 6 | descendantOrSelfList | `<<900000000000511003 \|Acceptability\| OR <<363698007 \|Finding site\| OR <<900000000000013009 \|Synonym\| OR <<900000000000006009 \|Defining relationship\|` | existing | snomed-expand-ecl-or; snomed-expand-ecl-or-chain |  |
| 7 | descendantOfMemberOf | `<^900000000000538005 \|Description format reference set\|` | existing | snomed-expand-ecl-operator-on-memberOf-descendants | < ^ refset, adapted to the anatomy structure-and-part refset which has active members with descendants. |
| 8 | descendantOrSelfOfMemberOf | `<<^900000000000538005 \|Description format reference set\|` | existing | snomed-expand-ecl-operator-on-memberOf | << ^ refset. |
| 9 | descendantOrSelfAny | `<< *` | new | snomed-expand-ecl-descendant-or-self-of-any | count=0. |
| 10 | childOfAny | `<!*` | new | snomed-expand-ecl-child-of-any | count=0. |
| 11 | childOrSelfOfAny | `<<! *` | new | snomed-expand-ecl-child-or-self-of-any | count=0. |
| 12 | ancestorOrSelfOfAny | `>> *` | new | snomed-expand-ecl-ancestor-or-self-of-any | count=0. |
| 13 | topOf | `!!> (< 900000000000443000 \|Module\|)` | existing | snomed-expand-ecl-top-of-set, -top-of-set-children | !!> (< X) = the children of X. |
| 14 | topOfAny | `!!> *` | new | snomed-expand-ecl-top-of-any | 11 roots in the test edition (see 5). |
| 15 | bottomOf | `!!< (< 900000000000443000 \|Module\|)` | existing | snomed-expand-ecl-bottom-of-set |  |
| 16 | bottomOfAnyTooCostly | `!!< *` | new | snomed-expand-ecl-bottom-of-any | Snow Owl refuses !!< * as too costly; the IG test expects the result (count=0). Does the IG want to allow a too-costly response? |
| 17 | selfAndSelf | `138875005 \|SNOMED CT Concept\| AND 138875005 \|SNOMED CT Concept\|` | new | snomed-expand-ecl-self-and-self |  |
| 18 | selfAndOtherWithCommaAsOperator | `138875005 \|SNOMED CT Concept\| , 9271000132107 \|Abbreviation\|` | new | snomed-expand-ecl-comma-conjunction | Comma as a conjunction at the expression level. |
| 19 | multipleAndOperatorsWithoutBrackets | `138875005 \|SNOMED CT Concept\| AND 138875005 \|SNOMED CT Concept\| AND 138875005 \|SNOMED CT Concept\|` | new | snomed-expand-ecl-and-chain |  |
| 20 | multipleOrOperatorsWithoutBrackets | `138875005 \|SNOMED CT Concept\| OR 138875005 \|SNOMED CT Concept\| OR 138875005 \|SNOMED CT Concept\|` | new | snomed-expand-ecl-or-chain |  |
| 21 | binaryOperatorAmbiguityAndOr | `138875005 \|SNOMED CT Concept\| AND 138875005 \|SNOMED CT Concept\| OR 138875005 \|SNOMED CT Concept\|` | new | snomed-expand-ecl-ambiguous-precedence-and-or | The existing ambiguous-precedence test covers OR then AND. |
| 22 | ambiguousOrOperatorInRefinement | `(<64572001 \|Disease (disorder)\|: 363698007 \|Finding site\| = 74281007 \|Myocardium structure\| OR 404684003 \|Clinical finding\|)` | new | snomed-expand-ecl-ambiguous-or-in-refinement | Adapted to real liver/disease concepts. |
| 23 | ambiguousOrOperatorInConstraint | `(404684003 \|Clinical finding (finding)\| OR <64572001 \|Disease (disorder)\|: 363698007 \|Finding site\| = 74281007 \|Myocardium structure\|)` | new | snomed-expand-ecl-or-with-refined-operand |  |
| 24 | refinementAttributeSubExpression | `<{DRUG_ROOT}: (127489000 \|Has active ingredient\| OR {HAS_BOSS}) = ({INGREDIENT1} OR {INGREDIENT2})` | new | snomed-expand-ecl-refinement-attribute-and-value-sets | Adapted to the medicinal products in the test edition (Has active ingredient / Has precise active ingredient; halibut liver oil / barium sulfate). |
| 25 | refinementAttributeNotEquals | `<{DRUG_ROOT}: 127489000 \|Has active ingredient\| != {INGREDIENT1}` | new | snomed-expand-ecl-refinement-not-equals |  |
| 26 | refinementAnyAttributeNotEquals | `*: 127489000 \|Has active ingredient\| != {INGREDIENT1}` | new | snomed-expand-ecl-refinement-not-equals-any |  |
| 27 | refinementReversedAttributeEquals | `<105590001 \|Substance\|: R 127489000 \|Has active ingredient\| = {TRIPHASIL_TABLET}` | new | snomed-expand-ecl-refinement-reverse |  |
| 28 | refinementAnyAttributeName | `<105590001 \|Substance\|: R * = {TRIPHASIL_TABLET}` | new | snomed-expand-ecl-refinement-reverse-any-attribute |  |
| 29 | refinementAnyAttributeValue | `<105590001 \|Substance\|: R 127489000 \|Has active ingredient\| = *` | new | snomed-expand-ecl-refinement-reverse-any-value |  |
| 30 | refinementCardinalityZeroToUnbounded | `<{DRUG_ROOT}: [0..*] 127489000 \|Has active ingredient\| = <105590001 \|Substance\|` | new | snomed-expand-ecl-refinement-cardinality-zero-to-many |  |
| 31 | refinementCardinalityAtLeastOne | `<{DRUG_ROOT}: [1..*] 127489000 \|Has active ingredient\| = <105590001 \|Substance\|` | new | snomed-expand-ecl-refinement-cardinality-at-least-one |  |
| 32 | refinementCardinalityZeroToZeroEquals | `<{DRUG_ROOT}: [0..0] 127489000 \|Has active ingredient\| = <105590001 \|Substance\|` | new | snomed-expand-ecl-refinement-cardinality-zero |  |
| 33 | refinementCardinalityZeroToZeroNotEquals | `<{DRUG_ROOT}: [0..0] 127489000 \|Has active ingredient\| != <105590001 \|Substance\|` | new | snomed-expand-ecl-refinement-cardinality-zero-not-equals |  |
| 34 | refinementCardinalityZeroToZeroNotEqualsToSingleValue | `<{DRUG_ROOT}: [0..0] 127489000 \|Has active ingredient\| != {INGREDIENT1}` | new | snomed-expand-ecl-refinement-cardinality-zero-not-equals-single |  |
| 35 | refinementCardinalityZeroToOne | `<{DRUG_ROOT}: [0..1] 127489000 \|Has active ingredient\| = <105590001 \|Substance\|` | new | snomed-expand-ecl-refinement-cardinality-zero-to-one |  |
| 36 | refinementCardinalityOneOrTwo | `<{DRUG_ROOT}: [1..2] 127489000 \|Has active ingredient\| = <105590001 \|Substance\|` | new | snomed-expand-ecl-refinement-cardinality-one-to-two |  |
| 37 | refinementWithAttributeConjunction | `<{DRUG_ROOT}: 127489000 \|Has active ingredient\| = {INGREDIENT1}, 127489000 \|Has active ingredient\| = {INGREDIENT2}` | existing | snomed-expand-ecl-refinement-and | Ungrouped conjunction written with AND; the comma form is in -refinement-* tests throughout. |
| 38 | refinementWithAttributeDisjunction | `<{DRUG_ROOT}: 127489000 \|Has active ingredient\| = {INGREDIENT2} OR 127489000 \|Has active ingredient\| = {INGREDIENT4}` | existing | snomed-expand-ecl-refinement-or |  |
| 39 | refinementWithConjunctionAndDisjunction | `<{DRUG_ROOT}: 127489000 \|Has active ingredient\| = {INGREDIENT2} OR (127489000 \|Has active ingredient\| = {INGREDIENT4} AND 127489000 \|Has active ingredient\| = {INGREDIENT2})` | new | snomed-expand-ecl-refinement-nested-and-or |  |
| 40 | refinementAmbiguityAndOr | `<{DRUG_ROOT}: 127489000 \|Has active ingredient\| = {INGREDIENT2} AND 127489000 \|Has active ingredient\| = {INGREDIENT4} OR 127489000 \|Has active ingredient\| = {INGREDIENT2}` | new | snomed-expand-ecl-refinement-ambiguous-and-or |  |
| 41 | refinementAmbiguityOrAnd | `<{DRUG_ROOT}: 127489000 \|Has active ingredient\| = {INGREDIENT2} OR 127489000 \|Has active ingredient\| = {INGREDIENT4} AND 127489000 \|Has active ingredient\| = {INGREDIENT2}` | new | snomed-expand-ecl-refinement-ambiguous-or-and |  |
| 42 | refinementReversedCardinalityZeroToUnbounded | `<105590001 \|Substance\|: [0..*] R 127489000 \|Has active ingredient\| = {TRIPHASIL_TABLET}` | new | snomed-expand-ecl-refinement-reverse-cardinality |  |
| 43 | dotted | `{PANADOL_TABLET}.127489000 \|Has active ingredient\|` | new | snomed-expand-ecl-dotted-single-concept |  |
| 44 | dottedWithComplexLeftSide | `(<{DRUG_ROOT} MINUS {TRIPHASIL_TABLET}).127489000 \|Has active ingredient\|` | new | snomed-expand-ecl-dotted-complex-left-side |  |
| 45 | refinementGroupCardinalityTwoToTwo | `<{DRUG_ROOT}: [2..2] {127489000 \|Has active ingredient\| = <105590001 \|Substance\|}` | new | snomed-expand-ecl-group-cardinality-two | Over diseases with two liver finding-site groups. |
| 46 | refinementGroupCardinalityZeroToUnbounded | `<{DRUG_ROOT}: [0..*] {127489000 \|Has active ingredient\| = <105590001 \|Substance\|}` | new | snomed-expand-ecl-group-cardinality-zero-to-many |  |
| 47 | refinementGroupCardinalityZeroToZero | `<{DRUG_ROOT}: [0..0] {127489000 \|Has active ingredient\| = <105590001 \|Substance\|}` | new | snomed-expand-ecl-group-cardinality-zero |  |
| 48 | refinementGroupCardinalityZeroToOne | `<{DRUG_ROOT}: [0..1] {127489000 \|Has active ingredient\| = <105590001 \|Substance\|}` | new | snomed-expand-ecl-group-cardinality-zero-to-one |  |
| 49 | refinementWithGroupConjunction | `<{DRUG_ROOT}: [1..1] {127489000 \|Has active ingredient\| = <105590001 \|Substance\|, {HAS_BOSS} = <105590001 \|Substance\|}` | new | snomed-expand-ecl-group-cardinality-with-conjunction |  |
| 50 | refinementWithZeroToOneCardinalityInAttributeConjuction | `<<{DRUG_ROOT}: {[0..1] 127489000 \|Has active ingredient\| = <<105590001 \|Substance\|, [1..1] {HAS_BOSS} = <<105590001 \|Substance\|, [1..1] {HAS_TRADE_NAME} = <<105590001 \|Substance\|}` | new | snomed-expand-ecl-group-mixed-attribute-cardinalities | Adapted to disease finding site / morphology. |
| 51 | refinementWithGroupDisjunction | `<{DRUG_ROOT}: [1..1] {127489000 \|Has active ingredient\| = {INGREDIENT5} OR {HAS_BOSS} = {INGREDIENT5}}` | new | snomed-expand-ecl-group-cardinality-with-disjunction |  |
| 52 | refinementGroupNestedConjunctionDisjunction | `<{DRUG_ROOT}: [1..1] {(127489000 \|Has active ingredient\| = {INGREDIENT5} AND {HAS_BOSS} = {INGREDIENT6}) OR 127489000 \|Has active ingredient\| = {INGREDIENT6}}` | new | snomed-expand-ecl-group-nested-and-or |  |
| 53 | refinementStringEquals | `<{DRUG_ROOT}: {HAS_TRADE_NAME} = 'PANADOL'` | consult |  | String concrete values: SNOMED International has no string-valued attributes, so there is nothing in the test edition to query. Would need a synthetic attribute in the TEST module. Note Snow Owl writes strings in single quotes; the ECL grammar uses double quotes. |
| 54 | refinementStringManyValuedEquals | `<{DRUG_ROOT}: {HAS_TRADE_NAME} = ('PANADOL' 'TRIPHASIL')` | consult |  | As 53. |
| 55 | refinementAnyStringNotEquals | `*: {HAS_TRADE_NAME} != 'PANADOL'` | consult |  | As 53. |
| 56 | refinementIntegerEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} = #500` | new | snomed-expand-ecl-concrete-integer-equals | Hand-computed from sct2_RelationshipConcreteValues (4 products, integer values #1, #600, #4000). |
| 57 | refinementIntegerEqualsNegative | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} = #-500` | new | snomed-expand-ecl-concrete-integer-negative | No negative values in the test edition: parses, 0 results. |
| 58 | refinementIntegerNotEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} != #500` | new | snomed-expand-ecl-concrete-integer-not-equals | 0 results: every count of base of active ingredient is 1. |
| 59 | refinementDecimalEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} = #5.5` | consult |  | Decimal concrete values: the test edition has no decimal values (only #1, #600, #4000). Adding a product with a decimal strength to the TEST module would allow these, and would also let the IG decide the Snow Owl claim that integer and decimal comparisons never match each other (does #5 match #5.0?). |
| 60 | refinementDecimalEqualsNegative | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} = #-5.5` | consult |  | As 59. |
| 61 | refinementDecimalNotEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} != #5.5` | consult |  | As 59. |
| 62 | refinementIntegerLessThan | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} < #1` | new | snomed-expand-ecl-concrete-integer-less-than |  |
| 63 | refinementDecimalLessThan | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} < #1.0` | consult |  | As 59. |
| 64 | refinementIntegerLessThanOrEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} <= #1` | new | snomed-expand-ecl-concrete-integer-less-than-or-equals |  |
| 65 | refinementDecimalLessThanOrEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} <= #1.0` | consult |  | As 59. |
| 66 | refinementIntegerGreaterThan | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} > #1` | new | snomed-expand-ecl-concrete-integer-greater-than |  |
| 67 | refinementDecimalGreaterThan | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} > #1.0` | consult |  | As 59. |
| 68 | refinementIntegerGreaterThanEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} >= #1` | new | snomed-expand-ecl-concrete-integer-greater-than-or-equals |  |
| 69 | refinementDecimalGreaterThanEquals | `<{DRUG_ROOT}: {PREFERRED_STRENGTH} >= #1.0` | consult |  | As 59. |
| 70 | descendantsOfAllMembersOfReferenceSet | `<^900000000000538005 \|Description format reference set\|` | new | snomed-expand-ecl-operator-on-memberOf-descendants |  |
| 71 | defaultGroupCardinalityWithZeroToZeroAttributeCardinality | `<{DRUG_ROOT}: { [0..0] 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-attribute-cardinality-zero | SEMANTICS OPEN: Snow Owl expects { [0..0] A = X } to match nothing; the IG test encodes the reading in which a group without the attribute satisfies [0..0]. Needs a SNOMED Computable Languages Group ruling. |
| 72 | defaultGroupCardinalityWithZeroToZeroAttributeCardinalityNotEquals | `<{DRUG_ROOT}: { [0..0] 127489000 \|Has active ingredient\| != {INGREDIENT5} }` | new | snomed-expand-ecl-group-attribute-cardinality-zero-not-equals |  |
| 73 | defaultGroupCardinalityWithZeroToOneAttributeCardinality | `<{DRUG_ROOT}: { [0..1] 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-attribute-cardinality-zero-to-one |  |
| 74 | groupCardinalityZeroToZeroWithDefaultAttributeCardinality | `<{DRUG_ROOT}: [0..0] { 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-cardinality-zero | Same ECL shape as 47. |
| 75 | groupCardinalityZeroToOneWithDefaultAttributeCardinality | `<{DRUG_ROOT}: [0..1] { 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-cardinality-zero-to-one | Same ECL shape as 48. |
| 76 | groupCardinalityZeroToOneWithDefaultAttributeCardinalityNotEquals | `<{DRUG_ROOT}: [0..1] { 127489000 \|Has active ingredient\| != <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-cardinality-zero-to-one-not-equals |  |
| 77 | groupCardinalityZeroToZeroWithZeroToZeroAttributeCardinality | `<{DRUG_ROOT}: [0..0] { [0..0] 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-and-attribute-cardinality-zero-zero | SEMANTICS OPEN, as 71. |
| 78 | groupCardinalityZeroToOneWithZeroToZeroAttributeCardinality | `<{DRUG_ROOT}: [0..1] { [0..0] 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-and-attribute-cardinality-zero-one-zero-zero | SEMANTICS OPEN, as 71. |
| 79 | groupCardinalityZeroToZeroWithZeroToOneAttributeCardinality | `<{DRUG_ROOT}: [0..0] { [0..1] 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-and-attribute-cardinality-zero-zero-zero-one |  |
| 80 | groupCardinalityZeroToOneWithZeroToOneAttributeCardinality | `<{DRUG_ROOT}: [0..1] { [0..1] 127489000 \|Has active ingredient\| = <105590001 \|Substance\| }` | new | snomed-expand-ecl-group-and-attribute-cardinality-zero-one-zero-one |  |
| 81 | concept_activeOnly | `* {{ c active = true }}` | existing | snomed-expand-ecl-concept-filter-active-not-equals | {{ C active = true }} on * is the whole edition; the != false form on a subtree covers the shape. |
| 82 | concept_inactiveOnly | `* {{ c active = false }}` | existing | snomed-expand-ecl-concept-filter-inactive |  |
| 83 | concept_active_notequals | `* {{ c active != false }}` | new | snomed-expand-ecl-concept-filter-active-not-equals |  |
| 84 | term | `* {{ term = "Clin find" }}` | existing | snomed-expand-ecl-description-filter-term |  |
| 85 | termLessThanTwoChars | `* {{ term = "C" }}` | consult |  | Snow Owl rejects a match term shorter than 2 characters. The ECL specification sets no minimum; server-specific, not ported. |
| 86 | termDisjunction | `* {{ term = (match:"gas" wild:"*itis")}}` | new | snomed-expand-ecl-description-filter-term-match-and-wild |  |
| 87 | termMatchSynonymsDisabled | `* {{ term = "history" }}` | consult |  | Checks that Snow Owl's configured search synonyms do not apply to ECL term matching. Server-specific, not ported. |
| 88 | termWildCaseInsensitive | `* {{ term = wild:"*history*" }}` | new | snomed-expand-ecl-description-filter-wild-case-insensitive |  |
| 89 | termWildAnyCharacterShouldNotCreateQueryClause | `* {{ term = wild:"**" }}` | new | snomed-expand-ecl-description-filter-wild-any |  |
| 90 | multiDomainQueryAnd | `* {{ c active=false }} AND * {{ d term="clin find" }}` | new | snomed-expand-ecl-filters-across-and |  |
| 91 | multiDomainQueryOr | `* {{ c active=false }} OR * {{ d term="clin find" }}` | new | snomed-expand-ecl-filters-across-or |  |
| 92 | multiDomainQueryExclusion | `* {{ c active=false }} MINUS * {{ d term="clin find" }}` | new | snomed-expand-ecl-filters-across-minus |  |
| 93 | profile_default | `<< 105590001 \|Substance\| {{ + HISTORY }}` | new | snomed-expand-ecl-history-default | Hand-computed. The test edition carries the REPLACED BY association only (two active rows); SAME AS, WAS A, POSSIBLY/PARTIALLY EQUIVALENT TO are absent, so MIN/MOD/MAX cannot be told apart except through REPLACED BY. Recommend adding association rows for the other refsets to the TEST module. The default profile is taken to be HISTORY-MAX, as Snow Owl does - to confirm against the specification. |
| 94 | profile_all_association_refsets | `<< 105590001 \|Substance\| {{ + HISTORY (*) }}` | new | snomed-expand-ecl-history-all-refsets | Hand-computed. |
| 95 | profile_min | `<< 105590001 \|Substance\| {{ + HISTORY-MIN }}` | new | snomed-expand-ecl-history-min | Hand-computed: SAME AS is absent, so nothing is added. |
| 96 | profile_mod | `<< 105590001 \|Substance\| {{ + HISTORY-MOD }}` | new | snomed-expand-ecl-history-mod | Hand-computed. |
| 97 | profile_max | `<< 105590001 \|Substance\| {{ + HISTORY-MAX }}` | new | snomed-expand-ecl-history-max | Hand-computed. |
| 98 | profile_explicit_refsets | `<< 105590001 \|Substance\| {{ + HISTORY (900000000000526001 \|REPLACED BY association reference set\| OR 1186924009 \|PARTIALLY EQUIVALENT TO association reference set\|) }}` | new | snomed-expand-ecl-history-explicit-refset | Hand-computed, over << 4896000 |Toxic hepatitis| and its REPLACED BY predecessor from the TEST module. |

Totals: 11 existing, 74 new, 14 consult.

## Questions for the community

1. **Empty / whitespace-only ECL** (rows 1, 2): error or empty expansion? The grammar requires an expression constraint.
2. **`{ [0..0] A = X }` and the group + attribute cardinality combinations** (rows 71, 77, 78): Snow Owl says these match nothing; the IG tests encode the reading in which a role group that lacks the attribute satisfies `[0..0]`. This needs a ruling from the SNOMED Computable Languages Group; the expected files follow whichever reading is chosen.
3. **Decimal concrete values** (rows 59-61, 63, 65, 67, 69): the test edition has integer values only (#1, #600, #4000). A product with a decimal strength in the TEST module would allow these and settle whether `#5` and `#5.0` compare equal (Snow Owl says integer and decimal comparisons never match each other).
4. **String concrete values** (rows 53-55): SNOMED International has no string-valued attributes; these need a synthetic attribute in the TEST module, or should be dropped. Snow Owl also writes string values in single quotes, which is not ECL.
5. **History supplements** (rows 93-98): the test edition has the REPLACED BY association only. Adding SAME AS / WAS A / POSSIBLY EQUIVALENT TO / PARTIALLY EQUIVALENT TO rows to the TEST module would make MIN, MOD and MAX distinguishable. The default profile (`{{ + HISTORY }}`) is taken to be MAX - confirm.
6. **`!!< *` as too costly** (row 16): Snow Owl refuses it; the IG test expects the leaves (count=0). Should the IG allow a cost-based refusal, and if so in what form?
7. **Roots of the subset** (rows 5, 14): `!!> *` returns 11 concepts, not 1, because the subset extraction omitted the is-a relationships of module and metadata concepts, and inactive concepts have none. Either accept that as a property of the test edition or add those relationships to the subset.
8. **Server-specific tests** (rows 85, 87: minimum term length, search-synonym configuration): not ported.
9. **`^` on a reference set of descriptions, and `^ [field]` selection** (added separately as `snomed-expand-ecl-memberOf-descriptions` / `-memberOf-field-selection`): both currently expect `UNSUPPORTED_ECL`; whether field selection on concept-valued fields is required is open.
