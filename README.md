# Terminology Ecosystem IG

This documents the HL7 terminology ecosystem - policies and test cases.

## The Test Suite

The test cases live in `tests/`, registered in `tests/test-cases.json`, and are published as part
of this IG in the package `hl7.fhir.uv.tx-ecosystem`.

You do not clone or download this repository to run the tests. The test runner is built into the
[FHIR validator](https://github.com/hapifhir/org.hl7.fhir.core/releases), and it fetches the
published test package for you:

```
java -jar validator_cli.jar txTests -tx {server url} -test-version 1.9.3 -output {folder}
```

* `-test-version` names the published version of **this test suite** - it is not a FHIR version.
  It defaults to `current`, which is the ci-build of master and changes whenever master changes,
  so pin a released version when you need results you can repeat and compare over time.
  Released versions are listed on the IG's
  [history page](https://hl7.org/fhir/uv/tx-ecosystem/history.html); the most recent is 1.9.3.
* The FHIR version tested is the server's own. The runner asks the server which version it speaks
  and runs the R4 or the R5 form of each test accordingly - there is no separate R4 test suite.
  See [R4 and the Test Cases](https://hl7.org/fhir/uv/tx-ecosystem/r4.html).
* Every option, the format of the test files, and the modes that select optional test sets, are
  documented on the [Test Cases](https://hl7.org/fhir/uv/tx-ecosystem/testcases.html) page.

Master moves: tests are added and corrected as servers and the specification develop. That is what
the releases are for - test against a pinned `-test-version`, and move to a newer one when it
suits you.

## FHIR Foundation Project Statement

* Maintainers: Grahame Grieve (as FHIR Product Director) + Reuben Daniels (as Deputy FHIR Product Director)
* Issues / Discussion: Create Issues here on GitHub. Discussion about (potential) issues, see [Zulip](https://chat.fhir.org/#narrow/channel/437028-Terminology-Service-Test-Cases). 
* License: [CCO - Creative Commons Public Domain](https://github.com/FHIR/fhir-tools-ig/blob/master/LICENSE.txt)
* Contribution Policy: Contributions to this IG are made under [the HL7 GOM](https://www.hl7.org/permalink/?GOM). This IG uses the standard HL7 ci-build for IGs, checking the generated QA. 
* Security Information: [see below](#security)
* Compliance Information: This content is used by/integrated with the base FHIR tools and is used across all versions of FHIR (R2-R5)

## Security

As an implementation guide that includes no active content, there's no direct security related content. 

For issues with the scripts that launch the publisher, see https://raw.githubusercontent.com/HL7/ig-publisher-scripts

If you think that there's security issues in the terminology ecosystem as documented here, you can report them here 
using GitHub's [standard security reporting framework](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability#privately-reporting-a-security-vulnerability), or you can email the [FHIR Director](mailto:fhir-director@hl7.org) directly.

If you think that there's a security issue with one of the servers that is part of the ecosystem, report it directly to the server.
