# Using old versions of ICF with FHIR

The International Classification of Functioning, Disability and Health (ICF) is published by the
World Health Organization. It classifies health and health-related domains - what a person's body
does, its structures, what the person does, and the environment they do it in - and is the WHO
framework for measuring health and disability.

ICF is now also published as a linearization of ICD-11 (`http://id.who.int/icd/release/11/icf`,
see [Using ICD-11 with FHIR](icd-11.html)). This page is about the older, standalone
releases of ICF - the 2001 classification and its annual updates - which use the familiar
alphanumeric codes such as `b1100` and `s7300`, and are still widely used. Two of them are
available as CodeSystems in the `fhir.tx.support` package: `2001a` and `2017a`.

## Summary

| | ICF (pre ICD-11) |
|---|---|
| **Source** | WHO |
| **System** | `http://hl7.org/fhir/sid/icf` |
| **OID** | `2.16.840.1.113883.6.254` |
| **Version** | The release: `2001a` or `2017a`. The version is needed - the content differs between releases. See [Versions](#versions) |
| **Code** | The category code, e.g. `b1100`, optionally followed by qualifiers: `s7300.183`, `d450.12`. See [ICF Codes](#icf-codes) and [Qualifiers](#qualifiers) |
| **Display** | The category's title; for a qualified code, the title followed by the qualifiers. See [Display Terms](#display-terms) |
| **Inactive** | The releases do not carry inactive codes |
| **Hierarchy** | A strict tree: component, chapter, block, then categories at up to four levels. Subsumption follows it, and a qualified code is subsumed by its category |
| **Qualifiers** | Yes - see [Qualifiers](#qualifiers) |
| **Filter Properties** | `concept` is-a etc., the code system's properties, and the qualifier filters - see [Filter Properties](#filter-properties) |
| **Implicit Value Sets** | None defined |

## License

ICF is published by WHO, and the same rules apply to licensing ICF as to licensing ICD. See
[http://icd.who.int/](http://icd.who.int/) and
[WHO's ICF pages](https://www.who.int/standards/classifications/international-classification-of-functioning-disability-and-health).

*Note: implementers should check WHO's current terms before using ICF.*

## ICF Codes

ICF has four components, and the first letter of every code says which one it belongs to:

| Letter | Component |
|---|---|
| `b` | Body functions |
| `s` | Body structures |
| `d` | Activities and participation |
| `e` | Environmental factors |

The letter is followed by the chapter number, then the second level (two more digits), then the
third and fourth levels (one digit each): `b1` Mental functions, `b110` Consciousness functions,
`b1100` State of consciousness. Every code also sits under its component in the hierarchy.

The code system also contains the components themselves (`b`, `s`, `d`, `e`), the chapters
(`b1`) and the blocks (`b110-b139`). These are headings: they are marked `notSelectable`, and
cannot be used to code with. In `2001a` the components are themselves under a root concept `ICF`.

```json
{
  "system" : "http://hl7.org/fhir/sid/icf",
  "version" : "2001a",
  "code" : "b1100",
  "display" : "State of consciousness"
}
```

## Qualifiers

In ICF itself, a code is not complete until it is qualified: the category says *what* is being
described, and qualifiers say how much of a problem there is, and of what kind. Qualifiers follow
the category code after a `.`, one digit each, and what each position means depends on the
component the code belongs to:

| Component | Position 1 | Position 2 | Position 3 | Position 4 |
|---|---|---|---|---|
| `b` Body functions | extent of impairment | | | |
| `s` Body structures | extent of impairment | nature of the change | localization | |
| `d` Activities and participation | performance | capacity (without assistance) | capacity with assistance (optional) | performance without assistance (optional) |
| `e` Environmental factors | *not supported yet - see below* | | | |

So `s7300.183` is *Structure of upper arm*, with a mild impairment (`1`), the nature not specified
(`8`), on both sides (`3`); and `d450.12` is *Walking*, with a mild performance problem (`1`) and a
moderate capacity problem (`2`).

In the FHIR ecosystem, though, an unqualified code has uses of its own, and the code system
accepts one as a valid code:

* value sets, concept maps and profiles name categories - which areas of functioning a form or
  a measure is about - rather than findings about a person
* the qualifier may be recorded somewhere other than the code: an Observation whose `code` is the
  category and whose `value` is the rating, or a questionnaire item asking about the category
* the hierarchy, `$lookup` and `$subsumes` work on categories

A record of a finding about a person should carry its qualifiers, and a value set can require
them - see [Filter Properties](#filter-properties).

The rules are:

* the qualifiers are positional, and a code may have as many as its component allows (above):
  one for `b`, up to three for `s`, and up to four for `d` - performance and capacity, then
  optionally capacity with assistance and performance without assistance. `b1100.21` and
  `d450.12345` are not valid
* a position cannot be skipped. `8` (not specified) is what fills a position that has nothing to
  say: to give a localization without a nature, write `s7300.183`, not `s7300.1 3`
* each digit must be one of the values of its position's scale (below): `b1100.5` is not valid,
  since the extent has no 5, and nor is `d450.15`
* only a selectable category can be qualified - not a component, chapter or block (`b1.2` and
  `b110-b139.2` are not valid)
* qualifiers are not required by the code system: `b1100` on its own is a valid code. Some uses
  of ICF do require them, and a value set can say so - see [Filter Properties](#filter-properties)

A qualified code is a code in its own right: `Coding.code` is the whole thing.

```json
{
  "system" : "http://hl7.org/fhir/sid/icf",
  "version" : "2001a",
  "code" : "s7300.183",
  "display" : "Structure of upper arm: MILD impairment, both sides"
}
```

The values of each scale are concepts in the code system, so they can be looked up and their
definitions read. Each scale has a heading concept (`q1` to `q4`), and its values are
`q1-0`, `q1-1` and so on. These are not codes to use on their own: they have a `kind` property
that says which scale they belong to (`extent`, `nature`, `localization` or `problem`), where
every ordinary ICF code has the kind `primary`.

| Scale | Used for |
|---|---|
| `q1` Extent of impairment | position 1 of `b` and `s` codes |
| `q2` Nature of the change in the respective body structure | position 2 of `s` codes |
| `q3` Localization | position 3 of `s` codes |
| `q4` Extent of problem | every position of `d` codes |

**Extent of impairment** (`q1`)

| Value | Meaning | Clinical threshold guidance |
|---|---|---|
| 0 | NO impairment | 0-4% of the time, or negligible impact |
| 1 | MILD impairment | 5-24% of the time, or slight, low impact |
| 2 | MODERATE impairment | 25-49% of the time, or distinct, medium impact |
| 3 | SEVERE impairment | 50-95% of the time, or major, high impact |
| 4 | COMPLETE impairment | 96-100% of the time, or total impact |
| 8 | Not specified | Used when information is insufficient to grade the severity |
| 9 | Not applicable | Used when it is inappropriate to apply the qualifier |

**Nature of the change in the respective body structure** (`q2`)

| Value | Meaning |
|---|---|
| 0 | no change in structure |
| 1 | total absence |
| 2 | partial absence |
| 3 | additional part |
| 4 | aberrant dimensions |
| 5 | discontinuity |
| 6 | deviating position |
| 7 | qualitative changes in structure, including accumulation of fluid |
| 8 | not specified |
| 9 | not applicable |

**Localization** (`q3`)

| Value | Meaning |
|---|---|
| 0 | more than one region |
| 1 | right |
| 2 | left |
| 3 | both sides |
| 4 | front |
| 5 | back |
| 6 | proximal |
| 7 | distal |
| 8 | not specified |
| 9 | not applicable |

**Extent of problem** (`q4`)

| Value | Meaning | Clinical threshold guidance |
|---|---|---|
| 0 | NO problem | none, absent, negligible: 0-4% |
| 1 | MILD problem | slight, low: 5-24% |
| 2 | MODERATE problem | medium, fair: 25-49% |
| 3 | SEVERE problem | high, extreme: 50-95% |
| 4 | COMPLETE problem | total: 96-100% |
| 8 | not specified | Used when information is insufficient to grade the severity |
| 9 | not applicable | Used when it is inappropriate to apply the qualifier |

**Environmental factors.** In ICF, an environmental factor (`e`) has one qualifier, which says
whether the factor is a barrier or a facilitator, and how much of one: a barrier is written with
`.` (`e310.2`, a moderate barrier), and a facilitator with `+` in place of the `.` (`e310+3`, a
substantial facilitator). There are open questions about the facilitator scale, and qualifiers on
environmental factors are not supported yet: `e310` is a valid code, but `e310.2` and `e310+3` are
not. Expect this to change.

## Versions

Two releases are available:

* `2001a` - the 2001 classification. It carries translations - see [Display Terms](#display-terms)
* `2017a` - the classification with the updates to 2017. English only

The version is needed (`versionNeeded` is true): a code may mean something different, or not
exist, in a different release, so a Coding should say which release it is from. The qualifier
scales are the same in both releases.

ICF in ICD-11 is a different code system, with different codes (`http://id.who.int/icd/release/11/icf`,
versioned by ICD-11 release), and is not a version of this one.

## Display Terms

The display of a code is the category's title. In `2001a`, translations are available in French
(`fr`), Spanish (`es`), Russian (`ru`), Chinese (`zh`) and Turkish (`tr`), and `displayLanguage`
(or the HTTP `Accept-Language`) selects one. `2017a` has English only.

The display of a qualified code is generated: the category's title, a `:`, then the meaning of
each qualifier, separated by `, `. The qualifiers of a `d` code are each introduced by what they
are - `performance`, `capacity`, and so on - since they share one scale. A qualifier of `8` (not specified) says nothing, so it is left
out - and if every qualifier is `8`, the display is just the category's title:

| Code | Display |
|---|---|
| `b1100.2` | State of consciousness: MODERATE impairment |
| `b1100.9` | State of consciousness: Not applicable |
| `s7300.183` | Structure of upper arm: MILD impairment, both sides |
| `s7300.81` | Structure of upper arm: total absence |
| `s7300.888` | Structure of upper arm |
| `d450.12` | Walking: performance MILD problem, capacity MODERATE problem |
| `d450.8834` | Walking: capacity with assistance SEVERE problem, performance without assistance COMPLETE problem |

The category's own title is not a valid display for a qualified code (unless every qualifier is
`8`): `$validate-code` of `s7300.183` with the display `Structure of upper arm` reports the wrong
display.

## Properties

These are the properties a `$lookup` may return:

| Property | Type | Applies to | Description |
|---|---|---|---|
| `inclusion` | string | categories | Inclusion criteria for the category |
| `exclusion` | string | categories | Exclusion criteria for the category |
| `parent` | code | all | The parent of the concept. For a qualified code, its category |
| `child` | code | all | A child of the concept. A qualified code has none |
| `notSelectable` | boolean | all | The concept is a heading (component, chapter, block, scale) and cannot be used to code with |
| `kind` | code | all | `primary` for an ICF code; `extent`, `nature`, `localization` or `problem` for a qualifier value or scale |
| `extent` | code | qualified `b` and `s` codes | The extent qualifier, as a scale value, e.g. `q1-2` |
| `nature` | code | qualified `s` codes | The nature qualifier, e.g. `q2-8` |
| `localization` | code | qualified `s` codes | The localization qualifier, e.g. `q3-3` |
| `performance` | code | qualified `d` codes | The performance qualifier, e.g. `q4-1` |
| `capacity` | code | qualified `d` codes | The capacity (without assistance) qualifier |
| `capacity-with-assistance` | code | qualified `d` codes | The capacity with assistance qualifier |
| `performance-without-assistance` | code | qualified `d` codes | The performance without assistance qualifier |

A qualified code has its category's properties as well, other than `parent`. A qualifier that is
`8` is still reported: `s7300.183` has `nature` `q2-8`.

## Filter Properties

| Property | Operators | Values | Description |
|---|---|---|---|
| `concept` | `is-a`, `descendent-of`, `is-not-a`, `child-of`, `=`, `in`, `regex` | codes | The hierarchy. A qualified code is in the results when its category is: `is-a b1` includes `b1100.2` |
| `extent`, `nature`, `localization`, `performance`, `capacity`, `capacity-with-assistance`, `performance-without-assistance` | `exists` | `true`, `false` | Whether the code must have this qualifier (`true`) or must not (`false`). A qualifier of `8` counts as absent: it never satisfies `true`, and always satisfies `false`. A qualifier the code's component does not have is absent, so `extent exists true` excludes every `d` code |
| `qualified` | `=` | `true`, `false` | Qualified codes only, or none at all. `s7300.888` is qualified, though it says nothing |
| `kind`, `notSelectable`, ... | `=`, `in`, `not-in`, `regex` | | The code system's other properties |

The qualifier filters combine with the others in an include, so a value set can say which codes,
and which qualifiers. For example, body structures, which must have all three qualifiers:

```json
"compose" : {
  "include" : [{
    "system" : "http://hl7.org/fhir/sid/icf",
    "version" : "2001a",
    "filter" : [
      { "property" : "concept", "op" : "is-a", "value" : "s" },
      { "property" : "extent", "op" : "exists", "value" : "true" },
      { "property" : "nature", "op" : "exists", "value" : "true" },
      { "property" : "localization", "op" : "exists", "value" : "true" }
    ]
  }]
}
```

`s7300.123` is in this value set; `s7300.12` (no localization), `s7300.128` and `s7300.183` (an
`8`) and `s7300` are not.

Or activities and participation, which must have a capacity qualifier - `d450.12` is in it, and
`d450.1` and `d450.18` are not:

```json
"compose" : {
  "include" : [{
    "system" : "http://hl7.org/fhir/sid/icf",
    "version" : "2001a",
    "filter" : [
      { "property" : "concept", "op" : "is-a", "value" : "d" },
      { "property" : "capacity", "op" : "exists", "value" : "true" }
    ]
  }]
}
```

And all of ICF, with no qualified codes:

```json
"compose" : {
  "include" : [{
    "system" : "http://hl7.org/fhir/sid/icf",
    "version" : "2001a",
    "filter" : [{ "property" : "qualified", "op" : "=", "value" : "false" }]
  }]
}
```

A value set that lists codes contains only the codes it lists: if it lists `b1100.2`, it does not
contain `b1100.3` - or `b1100` - and if it lists `s7300.888`, it does not contain `s7300.88`, even
though the two say the same thing.

**Expansions.** Qualified codes can be listed in a value set, and are expanded with their
generated displays. But they cannot be enumerated: an include that admits qualified codes - any
include without `qualified = false` - expands to the plain codes it contains, and the expansion is
marked as incomplete (`valueset-unclosed`), since the qualified codes are not in it.

## Implicit Value Sets

No implicit value sets have been defined for ICF.
