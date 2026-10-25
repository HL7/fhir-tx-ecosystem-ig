
### Introduction

This IG defines the HL7 terminology ecosystem framework: a set of terminology servers that serve different
code systems on behalf of different code system authorities, and a coordination server that routes
each terminology request to the server that can answer it.

HL7 defines one instance of an ecosystem, the public one which is what HL7's tools use for
terminology - validators, the IG publisher, code generators, and profile and terminology authoring
tools. 

Other authorities and networks maybe define their own terminology ecosystems using the same framework. 

This specification has two jobs. It sets out what a server has to do to be part of the ecosystem, with the test
cases that check it; and, because the tools depend on getting the same answer from every server, it
pins down how the terminology operations behave wherever the FHIR specification leaves room for
interpretation - which messages are returned, how inactive codes, versions, languages and
post-coordinated expressions are handled, and so on. The second job is now the larger one: the
[requirements](requirements.html) and the test cases are, in effect, a detailed operational
specification of the FHIR terminology services.

The test cases in this specification are maintained in association with the terminology authorities that publish 
the terminologies that are tested, and in many cases, serve to define the behaviour of the terminologies that 
are tested. In some cases, they are jointly maintained and official:

* The Snomed CT tests are jointly maintained by the [Snomed International Computed Languages](https://conf.spaces.snomed.org/wiki/spaces/SLPG/overview) Project Group and the HL7 community, and are considered the official test cases for ECL in particular
* The ICD-11 test cases are jointly maintained by [WHO](https://icd.who.int/en/) and the HL7 community, and are authoritative for ICD-11. The WHO FHIR API implementation conforms to these tests, as must other recognized ICD-11 servers

The LOINC and OMOP tests may become official; this is work-in-progress.

#### Who this is for

* **Server implementers**: the [Requirements for Servers](requirements.html) and
  [Language support](languages.html), the [Test Case Documentation](testcases.html) - how to run the
  tests, and how they work - and the [Test Case Registry](test-registry.html)
* **Server operators**: what it takes to become an [Approved Server](approved-servers.html), and how
  servers are registered in the [Ecosystem](ecosystem.html)
* **Tool developers**: the [Ecosystem](ecosystem.html) definition - the registries, and how requests
  are routed to the right server

#### Status

The current release of the test cases is **1.10.0** (25 October 2026). The
[Test Case History](test-history.html) lists what changed in each release. Test against a released
version: the default, `current`, is the ci-build, and changes whenever the tests do.

| Area | Status |
| --- | --- |
| The core operations - `$expand`, `$validate-code`, `$lookup`, `$translate`, `$subsumes` - with versions and languages | Stable. Releases add tests and correct expected responses, but the rules change rarely |
| SNOMED CT post-coordination | Settling. The scope servers have to support was agreed by the SNOMED Computable Languages Group in October 2026 (see the [requirements](requirements.html)), and the tests are being written to it |
| `$closure`, and [features](testcases.html#features) | New in 1.10.0 |
| ICD-11 and ICF | Draft. The [ICD-11](icd-11.html) and [ICF](icf.html) notes will move to terminology.hl7.org in time |
| The `tx.fhir.org` mode | Tests specific to tx.fhir.org. No other server is expected to pass them |

The test cases will be migrated to TestPlan and matchetypes at some stage. They are written for R5,
and also run against R4 servers (see [R4 and the Test Cases](r4.html)).

The test cases in the current build, by [mode](testcases.html#modes):

{% json tests/test-cases.json liquid/summary.liquid %}

#### Taking part

A server becomes an [Approved Server](approved-servers.html) by passing the test cases for the
modes it claims, against a released version, on a public test endpoint, and being approved by the
FHIR Product Director. Problems with the test cases, or with the requirements, 
are firstly discussed on [the Terminology Service Test Cases at chat.fhir.org](https://chat.fhir.org/#narrow/channel/437028-Terminology-Service-Test-Cases),
and then can be raised as issues or PRs on the [IG's repository](https://github.com/HL7/fhir-tx-ecosystem-ig). Changes to the tests
are worked through with the authors of the servers, and agreed with the FHIR Product Director. 

This specification is maintained by Grahame Grieve, as FHIR Product Director, and Reuben Daniels, as Deputy FHIR
Product Director. Contributions are made under the [HL7 GOM](https://www.hl7.org/permalink/?GOM), and
the IG is built with the standard HL7 ci-build for IGs, which checks the generated QA. The license is in the
[repository](https://github.com/HL7/fhir-tx-ecosystem-ig/blob/main/LICENSE.txt). The content is used by, and built into, the base FHIR tools, which use the ecosystem for every version of FHIR after R4. 

#### Security

This IG contains no active content. If you think there is a security issue in the terminology ecosystem
as it is documented here, report it privately using GitHub's
[security reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability#privately-reporting-a-security-vulnerability)
on the [IG's repository](https://github.com/HL7/fhir-tx-ecosystem-ig), or email the
[FHIR Product Director](mailto:fhir-director@hl7.org). A security issue with one of the servers in the
ecosystem should be reported to that server's operators.


#### In this IG

* Requirements: [for Servers](requirements.html), and for [Language support](languages.html)
* Ecosystem: the [Ecosystem](ecosystem.html) of servers, and the [Approved Servers](approved-servers.html)
* Test Cases: [Documentation](testcases.html), [Version History](test-history.html),
  [R4 and the Test Cases](r4.html), and the [Registry](test-registry.html)
* Code system specific notes: [ICD-11](icd-11.html) and [ICF](icf.html) (draft - these will move to
  terminology.hl7.org in time)

