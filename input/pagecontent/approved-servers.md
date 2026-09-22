### Approved Servers

This page lists the terminology servers that have been approved for the
[HL7 terminology ecosystem](ecosystem.html).

#### What an approved server is

A server is **approved** when all three of the following are true:

1. **It passes the test cases.** The server passes the [test cases](testcases.html) for the
   [modes](testcases.html#modes) it claims, against a released version of the test cases. `general` is
   claimed by every server; the other modes say which code systems and features the server additionally
   supports, and a server is only held to a mode it claims.

2. **It has a public test endpoint.** There is an endpoint that anyone can point the test runner at, so
   that the claim can be checked by a third party rather than taken on trust:

   ````
   java -jar validator_cli.jar txTests -tx {endpoint} -test-version {version} -mode {modes}
   ````

   The endpoint has to be reachable without credentials and loaded with whatever content the claimed
   modes require - the SNOMED CT test subontology, for instance, for the `snomed` mode. It does not
   have to be a production service, and it does not have to be the endpoint that the server's operators
   use for anything else.

3. **The FHIR Product Director has approved it.** Approval is a review, not an automatic consequence of
   a passing test run: the Product Director checks which modes are claimed, what the test output
   actually shows, and that the test end-point is available.

Approval is about the **server** - the implementation, and the public endpoint that demonstrates it. It
is a different thing from being **registered** in the ecosystem, which is about a particular deployment
being [authoritative](ecosystem.html#authoritative-servers) for particular content, and is recorded in
the [server registries](ecosystem.html#the-server-registries) rather than here. A server has to be
approved before a deployment of it is registered, but an approved server may have no registered
deployments, and one approved server may have many.

#### Approved servers

`general` is claimed by every server on this list, so the modes column names only the modes beyond it.

| Server | Responsible organization | Test endpoint | Test cases version | Additional modes |
| --- | --- | --- | --- | --- |
| [FHIRsmith](https://github.com/HealthIntersections/FHIRsmith) | [Health Intersections Pty Ltd](http://www.healthintersections.com.au) | `https://tx.fhir.org/r4`, `https://tx.fhir.org/r5` | 1.9.4 | `snomed`, `omop`, `mimetypes`, `icd-11`, `closure` |
| [Ontoserver](https://ontoserver.app/site/) | [CSIRO Australian e-Health Research Centre](https://aehrc.csiro.au/) | `https://r4.ontoserver.csiro.au/fhir`, `https://r5.ontoserver.csiro.au/fhir` | 1.9.0 | `flat` |
| CATY *(to be confirmed)* | *to be confirmed* | *to be confirmed* | *to be confirmed* | *to be confirmed* |

Entries marked *to be confirmed* are awaiting confirmation from the server provider; see
[Maintaining this page](#maintaining-this-page) below.

Notes on the columns:

* **Server** links to the server's source repository where it has one, and to its product page
  otherwise
* **Test endpoint** is the public endpoint described above. Where a server offers more than one FHIR
  version, each is listed; the test runner determines the FHIR version from the endpoint itself, so
  each one is a separate run
* **Test cases version** is the version of *this IG* that the current release of the server passes -
  see [Versions of the test cases](testcases.html#versions-of-the-test-cases). It is not a FHIR
  version. A server passing an older version of the test cases than the current release is not thereby
  unapproved, but the gap is worth understanding
* **Additional modes** are the [modes](testcases.html#modes) the server claims beyond `general`. The
  `tx.fhir.org` mode is not listed: it gates tests that are specific to that one server, and no other
  server is expected to pass them

#### Maintaining this page

**An existing entry** is maintained by the server's provider. Providers keep their own row current -
particularly the test cases version and the modes, both of which move as the server is released - by
raising a pull request against `input/pagecontent/approved-servers.md` in the
[IG's repository](https://github.com/HL7/fhir-tx-ecosystem-ig). A provider does not need to ask before
correcting their own row.

**A new entry** starts with an email to <fhir-director@hl7.org> requesting approval of the
registration. Include enough to fill a row, and the evidence behind it:

* the server's name, and a link to its repository or product page
* the responsible organization, and a link
* the public test endpoint, and the FHIR version or versions it speaks
* the version of the test cases the server passes
* the modes claimed beyond `general`
* the test runner's output for that run - the summary line and, where anything did not pass, what and
  why

The Product Director reviews the request and, on approval, the row is added.

An entry is expected to stay current. An entry whose test endpoint stops responding, or whose provider
stops maintaining it, may be removed - after the provider has been asked.
