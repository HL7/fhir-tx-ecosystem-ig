Note: This page is temporarily here until it migrates to terminology.hl7.org

### Using ICD-11 with FHIR

ICD-11 is published by the World Health Organization as three code systems that share one
underlying content model:

* **MMS** - the Mortality and Morbidity Statistics linearization: the classification used for
  coding diagnoses, causes of death, and the like
* **ICF** - the International Classification of Functioning, Disability and Health
* **Foundation** - the WHO Family of International Classifications Foundation: the semantic
  network from which the linearizations are derived

#### Summary

| | MMS | ICF | Foundation |
|---|---|---|---|
| **Source** | WHO | WHO | WHO |
| **System** | `http://id.who.int/icd/release/11/mms` | `http://id.who.int/icd/release/11/icf` | `http://id.who.int/icd/entity` |
| **OID** | none assigned | none assigned | none assigned |
| **Version** | The release, `YYYY-MM` (e.g. `2026-01`) | The release, `YYYY-MM` | The release, `YYYY-MM` |
| **Code** | The entity id, e.g. `257068234`, or a postcoordinated expression of entity ids. See [ICD-11 Codes](#icd-11-codes) | The entity id, or an expression (with ICF's `.` qualifier) | The entity uri, e.g. `http://id.who.int/icd/entity/257068234` (not yet settled - see [ICD-11 Codes](#icd-11-codes)) |
| **Display** | The entity's title, in the requested language where WHO publishes one | As MMS | As MMS |
| **Inactive** | ICD-11 releases do not carry inactive codes | As MMS | As MMS |
| **Hierarchy** | A strict tree (chapters, blocks, windows, categories); subsumption follows it | A strict tree | A poly-hierarchy: an entity may have several parents |
| **Postcoordination** | Yes - see [Post Coordination Expressions](#post-coordination-expressions) | Yes | No |
| **Filter Properties** | `concept` is-a / descendent-of - see [Filter Properties](#filter-properties) | As MMS | As MMS |
| **Implicit Value Sets** | None defined yet - see [Implicit Value Sets](#implicit-value-sets) | | |

#### License

ICD-11 is licensed by WHO under the Creative Commons Attribution-NoDerivs 3.0 IGO license
(CC BY-ND 3.0 IGO); see the [ICD-11 license](https://icd.who.int/docs/icd-api/license) and the
full [ICD-11 Terms of Use and License Agreement](https://icd.who.int/en/docs/ICD11-license.pdf).

*Note: the ICD-11 license is subject to further revision at this time. Implementers should check
the WHO license page for the current terms.*

#### ICD-11 Codes

Every ICD-11 MMS or ICF entity has two identifiers:

* **the classification code** - the short code coders know, such as `1A00` (Cholera) or `XN8P1`.
  Groupers such as blocks have none, and a short code can change between releases.
* **the entity uri** - such as `http://id.who.int/icd/release/11/mms/257068234`. Every entity has
  one, and it does not change between releases. A residual category ("other" or "unspecified")
  has a derived uri such as `http://id.who.int/icd/release/11/mms/1363559646/other`.

FHIR uses the entity uri, in its short form: the uri is always the code system url + `/` + the
entity id, so it is split between `Coding.system` and `Coding.code`:

```json
{
  "system" : "http://id.who.int/icd/release/11/mms",
  "version" : "2026-01",
  "code" : "257068234",
  "display" : "Cholera"
}
```

The entity id is the code, and the normal form. The classification code is accepted on input as
an alternative spelling - `$validate-code` reports it as valid but not in normal form, and
returns the entity id as `normalized-code` - and is returned as the `classificationCode` property.
`$validate-code` also returns a `classificationCode` output parameter when the code was given with
entity ids, whether or not in normal form: it is the classification code of the normal form, so
for an expression written out of order it matches `normalized-code`. A code given with
classification codes gets none. The full entity uri is not a code in MMS or ICF.

The Foundation was not part of this decision: its codes are, for now, the full entity uri.

```json
{
   "resourceType": "Condition",
   "code": {
      "coding": [
         {
            "extension": [{
                  "url": "http://id.who.int/icd/schema/extension/classificationCode",
                  "valueCode": "1A00"
            }],
            "system": "http://id.who.int/icd/release/11/mms",
            "version": "2026-01",
            "code": "257068234",
            "display": "Cholera"
         }
      ]
   }
}
```

**The classificationCode extension.** Because the code is the entity id, a Coding by itself does
not carry the classification code users know. The extension
`http://id.who.int/icd/schema/extension/classificationCode` (provisional url) carries it beside
the id, as a `valueCode`: the short code for a simple code, or the short-code form of the
expression - the same expression with each part's classification code - for an expression. It
saves receiving software from looking up the code to show it. The extension is optional, and is
only defined for a Coding whose system is the MMS or ICF and whose code is an entity id or an
expression in id form. When it is present it is validated: a value that is not the code's
classification code, the extension on a code that has no classification code, on a code that is
already given as its classification code, or on a Coding in any other system - the Foundation
included, which has no classification codes - is an error, and the Coding is not valid.

#### Post Coordination Expressions

MMS and ICF codes may be postcoordinated: a stem code is refined by further codes. The syntax
has two separators and no spaces:

* `&` - a qualifier of the code before it, which must be a value on one of that code's
  postcoordination axes: `257068234&194483911` (1A00 Cholera, Vibrio cholerae O1, biovar cholerae)
* `/` - a cluster member: another code that qualifies the first (root) code of the expression,
  and may carry its own `&` qualifiers:
  `2110065613&271422288&2040998541/1847586521&1021954298&1670965842`
  (NA00.2 Superficial injury of ear, left, contusion / PG50 Fall … on the same level, stairs, home)

ICF also writes a qualifier with a `.`: `1860600370/unspecified.992904195` (d5409.qp3). The `.`
is an ICF-only rule, and is kept as written.

Everything after a `/` qualifies the root; there is no deeper nesting. The order of the codes
within a group, and the order of the `/` members, carries no meaning, but the grouping does: `&`
qualifiers belong to the code they follow. A `/` within a residual's id
(`1363559646/other`) is part of the id, not a separator.

The normal form fixes an order anyway: the root first; each group of qualifiers in the order the
qualified code defines its axes, and by entity id (string order) within an axis; the `/` members
the same way against the root's axes, with any member on no axis of the root last; duplicate
codes deleted. The normal form never changes `&` to `/` or the reverse.

See the ICD-11 Reference Guide,
[Extension codes and postcoordination](https://icdcdn.who.int/icd11referenceguide/en/html/index.html#extension-codes-and-postcoordination)
and
[Adding detail - postcoordination and cluster coding](https://icdcdn.who.int/icd11referenceguide/en/html/index.html#adding-detail-postcoordination-and-cluster-coding-with-multiple-stem-codes-and-extension-codes),
for WHO's account of postcoordination and cluster coding.

#### Versions

WHO publishes a new release of ICD-11 each year, and each release covers all three code systems.
The version is the release identifier, in the form `YYYY-MM` - for example `2026-01` - and is
the same for MMS, ICF and the Foundation in a given release. Entity ids are stable across
releases; classification codes may change.

#### Display Terms

The display for an entity is its title. Where WHO publishes a translation of the classification,
the title is available in that language, and `displayLanguage` (or the HTTP `Accept-Language`)
selects it. The other text WHO publishes for an entity - index terms, inclusions, coding notes,
definitions, fully specified names - is returned as designations, their `use` coded in
`http://id.who.int/icd/schema` (`indexTerm`, `inclusion`, `codingNote`, and so on).

Translations are WHO's: only the languages WHO publishes are available. A request for a language
WHO does not publish gets the English display, and `$lookup` returns a `warning` saying so.

The display of a postcoordinated expression is built from its parts: cluster members separated by
` / `, each qualifier in brackets after the code it qualifies -
`Cholera [Vibrio cholerae O1, biovar cholerae]`.

#### Supplements

Only WHO can define a global language pack for ICD-11: a translation of the classification
belongs to WHO's release, not to a code system supplement. Supplements defined by anyone else must
not present themselves as global language packs - they are local additions, and should say so,
so that local usage is clear to whoever receives the data.

#### Properties

These are the properties a `$lookup` may return:

| Property | Type | Applies to | Description |
|---|---|---|---|
| `code` | code | all | The code: the entity id (MMS, ICF) or uri (Foundation); for an expression, its normal form |
| `classificationCode` | code | MMS, ICF | The short classification code, or the short-code form of an expression, where there is one |
| `classKind` | code | MMS, ICF | `chapter`, `block`, `window` or `category` |
| `notSelectable` | boolean | MMS, ICF | The entity is a grouper and cannot be used to code with |
| `parent` | code | all | A parent of the entity |
| `child` | code | all | A child of the entity |
| `browserUrl` | string | all | The entity's page in the ICD-11 browser |
| `source` | Coding | MMS, ICF | The Foundation entity this entity was derived from (returned by default) - see below |
| `blockId` | string | MMS, ICF | The block identifier, for a block |
| `codeRange` | string | MMS, ICF | The range of codes a block covers, e.g. `1A00-1A0Z` |
| `stem` | Coding | expressions | The root of a postcoordinated expression |
| `postcoordination` | (subproperties) | MMS, ICF | One per postcoordination axis - see below |

**The Foundation link.** `source` is a Coding in the Foundation (`http://id.who.int/icd/entity`),
whose code is the Foundation entity uri, with the entity's title as its display. A request for
the property `source.details` returns `source` with what a `$lookup` of that Foundation entity
would return, as subproperties: `definition` (string), where it has one, and `parent` (code, a
Foundation entity uri; repeats, since the Foundation is not a tree); and the entity's designations,
as `designation` parts with the same language, use and value parts as a `$lookup`'s own
designations. The operation does not define designations inside a property, but a subproperty
(a code and a value) cannot carry a designation's language and use. `source.details` is not a
property of its own, and `*` does not imply it.

**Postcoordination axes.** Each `postcoordination` property describes one axis, and carries its
content as subproperties:

| Subproperty | Type | Description |
|---|---|---|
| `axis` | Coding | The axis, in `http://id.who.int/icd/schema` (e.g. `laterality`, `specificAnatomy`) |
| `value` | Coding | A code on the axis (repeats); for a `/` member, it carries the member's own `postcoordination` subproperties, one level down |
| `required` | boolean | The axis must be given a value |
| `allowMultipleValues` | Coding | `AllowAlways`, `NotAllowed` or `AllowedExceptFromSameBlock`, in `http://id.who.int/icd/schema/allowMultipleValues` |
| `valueSet` | string | The canonical of the ValueSet of the values permitted on the axis |

For a postcoordinated expression, there is one `postcoordination` property for each axis that
has a value, and one more, with only `value`s, for `/` members that are on no axis of the root.
By default only `axis` and `value` are returned. With the `$lookup` parameter
`post-coordination-metadata=true`, every axis the code declares is listed - valued or not - and
each carries `required`, `allowMultipleValues` and `valueSet` as well. A plain (not
postcoordinated) concept therefore lists its axes only when `post-coordination-metadata=true`.

#### Filter Properties

| Property | Operators | Values | Description |
|---|---|---|---|
| `concept` | `is-a`, `descendent-of` | an entity id (MMS, ICF) or uri (Foundation), or a classification code | The entity and all its descendants (`is-a`), or the descendants only (`descendent-of`), following the code system's hierarchy |

`$expand` also supports a text `filter`, which matches the titles, index terms, inclusions and
fully specified names in the display language (and English), and the start of the classification
code - so `XN8` finds `XN8P1` and `XN8KD`.

For example, the extension codes for substances:

```json
"compose" : {
  "include" : [{
    "system" : "http://id.who.int/icd/release/11/mms",
    "version" : "2026-01",
    "filter" : [{ "property" : "concept", "op" : "descendent-of", "value" : "1321407960" }]
  }]
}
```

#### Implicit Value Sets

No implicit value sets have been defined for ICD-11 yet.

ICD-11 itself defines a value set for every postcoordination axis of every stem code - the values
WHO permits on that axis - at
`http://id.who.int/icd/release/11/mms/{entity id}/postcoordinationScale/{axis}`
(e.g. `http://id.who.int/icd/release/11/mms/257068234/postcoordinationScale/infectiousAgent`).
These are WHO's value sets, not FHIR implicit value sets; their canonicals are the `valueSet`
subproperty of a `postcoordination` property.
