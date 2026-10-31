
The tests assume that the server can accept code systems on the fly. 
If servers do not accept code systems on the fly, server authors will have to 
consult the FHIR product director. Either way, servers that do should pass all the tests, but the FHIR product director 
will review the test outcomes in order to approve a server. 

The test cases are in version R5, but the tests will run against either an R4 or an R5 server. 
See [R4 and the Test Cases](r4.html)

Note that the test cases will be migrated to use TestPlan+matchetypes at some stage.

### Running the tests 

The tests can be run by any runner that processes the tests correctly, but the easiest way to 
execute the tests is to use the [standard Java FHIR Validator](https://github.com/hapifhir/org.hl7.fhir.core/releases).
You do not download the tests yourself - the runner fetches the published test package for you:

````
java -jar validator_cli.jar txTests -tx {server} [-test-version {version}] [-output {folder}] 
    [-externals {file}] [-mode {mode},{mode}] [-suite {suite}] [-filter {text}] [-input {folder}]
    [-tester {name}]
````

The parameters are:

* `-tx`: the URL of the server to test for conformance. This is the only required parameter
* `-test-version`: which published version of *these test cases* to run - see 
  [Versions of the test cases](#versions-of-the-test-cases) below. It defaults to `current`
* `-output`: the folder to write the results to; defaults to your temporary directory. It contains 
  `test-results.json`, and the actual response for each failed test, which you can compare against 
  the expected response in the `/tests` directory of this IG's package with a comparison tool of 
  your choice (winmerge, beyondCompare, etc)
* `-externals`: a file that allows the server to use its own messages - they don't have to match 
  the tx.fhir.org messages. Copy `tests/messages-tx.fhir.org.json` for the format
* `-mode`: see [Modes](#modes) below. Several modes can be passed, separated by commas, or by 
  repeating the parameter
* `-suite`: run only the named suite, rather than all of them
* `-filter`: run only the tests whose name contains this text
* `-input`: an additional folder of test cases to run as well as the published ones
* `-tester`: who is running the tests, for `TestReport.tester` in `report.json` (default 
  `HL7 Ecosystem Test Runner`); the modes are added after it

Note that the FHIR version to test is *not* a parameter: the runner asks the server which version 
of FHIR it speaks, and runs the R4 or the R5 form of each test accordingly. 
See [R4 and the Test Cases](r4.html).

### Test Output

The runner writes a running commentary to the console. It starts with a header that records
what is actually being run - the source of the tests and their version, the output directory,
the server URL, whether an externals file is in use, and the modes - and it is worth reading,
because most surprising results turn out to be a test version or a mode that isn't what you
thought it was. Then, for each suite, a `Group {name}` line followed by one line per test:

````
Group expansions
   -- expand-simple: Pass (0.21sec)
   -- expand-filter-regex: Fail (0.19sec)
    Excluded code not found: http://hl7.org/fhir/test#code2
````

A failing test is followed by an error line with the reason - the first difference the runner
found between the expected response and the actual one, or, if the test threw rather than
compared, the exception and its stack trace. At the end, a summary line:

````
[name] passed all 1631 HL7 terminology service tests (modes general+snomed+icd-11, tests v1.9.4, runner v6.10.4)
````

or, if there were failures, `failed {n} of {m} ...` followed by a `Failed Tests:` line listing
them as `{suite}/{test}`, comma separated. The summary names the modes, the version of the test
cases, and the version of the runner, because a result means nothing without all three.

Note that the name is taken from the CapabilityStatement

The output directory (`-output`, defaulting to your temporary directory) contains:

* `test.log`: everything that went to the console, in the output folder where the rest of the
  results are, so that a run is still readable after you have closed the terminal
  (validator 6.10.5 and later)
* `test-results.json`: the date of the run, and every suite and test with its status and, for
  failures, the message
* `report.json`: the same run as a FHIR `TestReport`, with a score and an overall pass/fail
* `expected/` and `actual/`: for each **failing** test, the expected response and the response
  the server actually gave, written as a pair of files with the same name so that they can be
  compared with a comparison tool of your choice (winmerge, beyondCompare, etc). Both sides are
  scrubbed and sorted the same way before they are written, so the diff shows the difference
  the runner objected to and not incidental ones. Passing tests write nothing - if a test
  passes, there is nothing to look at
* `actual/$versions.json`: the server's response to `$versions`, recorded on connection,
  whether or not anything failed
* `conversions/`: only when testing an R4 server. For each operation, the R5 resource and the R4
  resource that was actually sent or received, as `conversions/{r4|r5}/{suite}/{test}.{mode}.json`,
  so that a failure caused by the version conversion can be told apart from one caused by the
  server. See [R4 and the Test Cases](r4.html)

Note that the same test writes to the same file names every time it fails. If you run the tests
several ways - R4 and R5, with and without server caching - each run overwrites the last one's
diffs, and a test that fails in only one of them can leave no evidence behind. Use a different
output directory for each variant, or, in server mode, a separate run or the `label` parameter
described below.

### Running the tests in a pipeline 

The simplest way to run the tests in a build is the command line above: it exits with 0 if every
test passed and 1 if any test failed, so a CI step needs no output parsing. Always pass
`-test-version` in a pipeline. The default is `current`, which is the ci-build of this IG and
changes whenever its master branch changes - a build that uses it can go red because the tests
moved, not because your server did. Pin a released version, and change it deliberately. Pass
`-output` as well, and keep the directory as a build artefact: when the step fails, the `expected/`
and `actual/` pair for each failing test is the only thing that tells you what went wrong, and it
is gone with the build container otherwise.

A command line run reports the run as a whole: one exit code, and one `test-results.json` and
`report.json`. If you run the tests several ways - against your R4 and R5 endpoints, say, or with
different sets of modes - each run is a separate result, and nothing combines them: give each run
its own `-output` directory, and compare or combine the reports yourself.

If you want your own test framework to report one result per test - to see which tests are newly
failing, to track them over time, or just to get a useful failure message next to the test that
produced it - use the validator's server mode instead. Start the validator once as an HTTP server,
start a run, ask it to run one test at a time, and end the run (validator 7.0.4 and later):

````
java -jar validator_cli.jar server {port} -version 5.0

POST http://localhost:{port}/txTest/setup                       (body: see below)
GET  http://localhost:{port}/txTest?session={id}&suite={suite}&test={test}&modes={modes}&label={label}
POST http://localhost:{port}/txTest/teardown?session={id}
````

**Setup** takes a JSON object with the things that cannot change during a run, and returns the
`session` to pass to the other two calls:

````
{
  "server" : "http://localhost:8080/fhir",
  "test-version" : "1.9.6",
  "externals" : { ... },
  "folder" : "my-server-r4",
  "tester" : "My Server CI"
}
````

* `server`: the server to test. This is the only required property
* `test-version`: as for `-test-version` on the command line, and with the same default - pin it
  in a pipeline
* `externals`: the *content* of an externals file (in the format of 
  `tests/messages-tx.fhir.org.json`), as a JSON object - not a file name
* `folder`: names the run's output folder inside the validator's temporary directory - a simple
  name, not a path, defaulting to the server's host name
* `tester`: as for `-tester` on the command line

The validator loads the test cases and connects to the server during setup, so the response also
says what it found: the version of the test cases, the server's FHIR version and software, the
features it declares, and the output folder. As on the command line, the FHIR version is the
server's, not a setting: a run against an R4 server runs the R4 form of each test. To test the
same server's R4 and R5 endpoints, start a run for each.

**Each test** returns an `OperationOutcome`. If it has no issue with a severity of `error`, the 
test passed; otherwise `details.text` is the failure message - the same message the command line
would have logged. The test cases and the suite's setup resources are loaded once, and reused, so
the cost of starting a JVM and loading the test package is paid once, not once per test. Drive it
from whatever test framework you already use: enumerate the suites and tests from 
`test-cases.json` in this IG's package (at the version you are testing), and make one call per
test.

`modes` replaces the runner's mode set for that call: pass every mode your server supports, exactly
as you would on the command line. Without it, the default is every mode except `tx.fhir.org`,
which may not match your server. A test that doesn't run - it is disabled, or gated on a mode you
did not ask for, a feature the server does not declare, or a FHIR version the server is not - comes
back as an error saying why it didn't run, because asking for it was a mistake; but it is not
counted as a failure in the run's results. The simplest approach is to leave out the tests that
won't run when you enumerate them. `label` names a subfolder of the run's folder for that one
test's output: if you run the same test more than one way in a run - with and without server
caching, say - give each way its own `label` so that the diffs for a test that fails in only one
of them survive.

A suite marked `sequential` (`closure`) has tests that depend on each other: in server mode, it is
up to you to run its tests in the order they are listed, one at a time, after the other suites.

**Teardown** ends the run, and returns the two reports a command line run writes - 
`test-results` and `report` (the `TestReport`) - as a JSON object, and writes them to the run's
output folder as `test-results.json` and `report.json`. Tests that didn't run are not counted, and a
test that was run more than once counts once, with its last result. A run's results only cover
that run: as on the command line, several runs give several reports.

The older form of the test call, with `server` (and optionally `externals`, naming a file in the 
test package, and `folder`) in place of `session`, still works: the first call for a server starts
a run for it with the `current` test cases, and `POST /txTest/teardown?server={url}` ends it.

### Versions of the test cases

The test cases change: tests are added, and corrected, as servers and the specification develop. 
So this IG is released periodically, and each release is a fixed set of test cases that you can 
test against repeatedly and compare results over time. The releases are listed on the 
[test history page](test-history.html), and you choose one with `-test-version`:

````
java -jar validator_cli.jar txTests -tx http://localhost:8080/fhir -test-version 1.9.3 -output ./results
````

`-test-version` names a version of **this IG** - a version like `1.9.3`. It is not a FHIR version, 
and passing a FHIR version (`-test-version 4.0.1`) will fail, because there is no such release of 
the test cases.

If you do not pass `-test-version`, the runner uses `current`: the tests as they stand in the 
ci-build of the master branch, which changes whenever master changes. That is the right choice 
when you are working with the FHIR product director on new tests, and the wrong one when you need 
a stable measure of your server's conformance.

### Test Suites and Test Cases 

All the tests are registered in `tests/test-cases.json`, which contains a list of suites, 
each of which contains a list of tests. A suite may have:

* `name`: the name of the suite (used in the test output, and to run a single suite)
* `description`: what the suite is for
* `notes`, `todo`, `mode-note`: notes for the people maintaining the tests. The runner ignores them
* `setup`: a list of files containing the resources (code systems, value sets, concept maps) 
  that the server needs in order to run the tests in the suite. Servers that cannot accept 
  resources on the fly have to load these some other way
* `mode` / `modes`: the suite only runs when the runner is in that mode (see below)
* `feature`: the suite only runs when the server declares all of these features, a comma 
  separated list (see below)
* `version`: the suite only runs against servers of that FHIR version (see below)
* `disabled`: if true, the suite is not run at all
* `sequential`: if true, the tests in the suite depend on each other, and must be run in the 
  order they are listed, one at a time, after all the other suites have finished. The `closure` 
  suite is like this: each test adds to the closure table the previous tests built. (In server 
  mode, running them that way is up to the caller)

A test may have:

* `name`: the name of the test, unique within the suite
* `description`: what the test is checking. This is the documentation for the requirement 
  the test enforces, so it should say why the expected response is the correct one
* `explanation`: a longer account of why the test exists, where `description` isn't enough. The
  runner ignores it
* `operation`: one of `expand`, `validate-code`, `cs-validate-code`, `lookup`, `subsumes`, 
  `translate`, `compare`, `closure`, `batch`, `batch-validate`, `cs-batch-validate`, `metadata`, `term-caps`
  (`batch-validate` is `ValueSet/$batch-validate-code`, `cs-batch-validate` is `CodeSystem/$batch-validate-code`)
* `request`: the file containing the request parameters (not used by `metadata` / `term-caps`)
* `response`: the file containing the expected response
* `request:{mode}` / `response:{mode}`: an alternative request or response used when the 
  runner is in that mode - e.g. `response:flat` for servers that return a flat expansion
* `mode` / `modes`, `feature`, `version`, `disabled`: as for suites
* `http-code`: the expected HTTP status code, where it isn't 200 (e.g. `422`, or `4xx`)
* `lenient-display`: sets the `lenient-display-validation` parameter on a `validate-code` or 
  `cs-validate-code` request, so that the same request can be tested both ways
* `profile`: a file containing a Parameters resource with the expansion parameters to use
* `Accept-Language`: the language to ask for
* `header`: an additional HTTP header to send ( `name`, `value`, and optionally `mode`)

### FHIR Versions 

The test cases are written in R5, and most of them are the same for an R4 server, but 
some questions have different answers in different versions of FHIR. Where a test only 
makes sense for particular versions, the `version` property says which:

* `"version" : "4.0"`: only run this against an R4 server (matches `4.0`, and any version 
  starting `4.0.`, so `4.0.1` matches)
* `"version" : "!4.0"`: run this against anything except an R4 server

Where the same test applies to all versions but the *response* differs, the difference is 
marked in the expected response with `$optional$` or `$only$` (see below).

### Test Templates 

Expected responses are not compared literally - they are templates. A string in an expected 
response may be one of:

* `$$`: any value
* `$id$`, `$uuid$`, `$url$`, `$token$`, `$string$`, `$date$`, `$instant$`, `$semver$`: any 
  value of that kind. (`$string$` means any string that has no leading or trailing whitespace)
* `$version$`: the FHIR version of the server being tested. `$version$` is also substituted 
  into longer strings, so `"...|$version$"` works
* `$choice:a|b|c$`: any one of the listed values
* `$fragments:a|b$`: any string that contains all of the listed fragments (case insensitive). 
  This is how messages are checked where the exact wording is not fixed
* `$external:N$`: the string registered as `N` in the externals file, which lets a server 
  provide its own wording for a message. See `tests/messages-tx.fhir.org.json` for the format. 
  `$external:N:a|b$` also gives fragments to use when no externals file is provided

An object in an expected response may carry:

* `$optional-properties$`: an array of the names of properties that the server is allowed to 
  omit. `*` means all of them
* `$count-arrays$`: an array of the names of properties whose content is not checked - only 
  the number of entries in them
* `$optional$`: this object may be omitted (see below)
* `$only$`: this object is required in some versions or modes, and prohibited in the rest 
  (see below)

### $optional$ and $only$ 

`$optional$` marks an entry in an array that the server does not have to return. It is either 
a boolean, or a filter that says when it is optional:

* `"$optional$" : true`: always optional. Note that this must be a boolean - `"true"` as a 
  string is read as a mode name, and does nothing
* `"$optional$" : "{mode}"`: optional unless the runner is in that mode
* `"$optional$" : "!{mode}"`: optional only when the runner is *not* in that mode. e.g. 
  `"!tx.fhir.org"` means "every server except tx.fhir.org may leave this out"
* `"$optional$" : "version:4"`: optional when the server's FHIR version starts with `4`
* `"$optional$" : "warning:{message}"`: always optional, but the runner reports the message 
  when the content is missing

`$only$` is the counterpart: where `$optional$` only ever relaxes a requirement, `$only$` 
says that the entry belongs to exactly one version or mode. When its filter passes the entry 
is required, and when it does not, the entry must **not** be present at all. The filter 
grammar is the same as `$optional$`'s. So `"$only$" : "version:4"` marks content that an R4 
server must return and an R5 server must not.

### Modes 

Some tests, and some parts of expected responses, only apply to servers with particular 
characteristics, or to a particular server. These are marked with a mode, and only run 
when that mode is passed to the runner with the `-mode` parameter. `general` is always 
on unless it is turned off with `-mode !general`. The modes in use are:

* `general`: the tests every server is expected to pass. This is the default
* `snomed`: servers that support SNOMED CT, loaded with the test subontology described in 
  `tx-source/readme.md`
* `omop`: servers that support OMOP
* `icd-11`: servers that support ICD-11
* `icf`: servers that support ICF (`http://hl7.org/fhir/sid/icf`) with its qualifier grammar - a category followed by `.` and the qualifiers its component takes (extent for `b`; extent, nature, localization for `s`; performance, capacity and two optional assistance qualifiers for `d`; a barrier after `.` or a facilitator after `+` for `e`) - and the qualifier `exists` and `qualified` `=` value set filters. The tests use version `2001a`
* `mimetypes`: servers that support the mime types code system (BCP 13, `urn:ietf:bcp:13`)
* `loinc`: servers that support LOINC, with the 2.83 release and its accessory files loaded
* `tx.fhir.org`: tests that are specific to tx.fhir.org - either its own bugs, or operations 
  that are still being trialled there. No other server is expected to pass these
* `flat`: servers that return a flat expansion rather than a hierarchical one. This mode 
  selects the `response:flat` alternative for the tests that have one

A mode name can also appear in `$optional$` / `$only$` in an expected response, so that a 
particular server (or every server but one) is held to a different requirement than the rest 
- `"$optional$" : "!tx.fhir.org"` is used throughout for content that only tx.fhir.org is 
required to return.

### Features

Modes are chosen by whoever runs the tests. Features are not: they come from what the server 
says about itself in its CapabilityStatement, which the runner reads before it runs anything. A 
suite or test with a `feature` only runs when the server declares every feature in it - 
`"feature" : "expand,tx-resource"` needs both. A suite or test that doesn't run because of its 
features is neither passed nor failed; the runner logs the features it found at the start, and 
lists them in the TestReport (as `TestReport.parameter.name`). The features are:

* `expand`, `lookup`, `subsumes`, `translate`, `closure`, `related`: the server declares that 
  operation - anywhere in the CapabilityStatement, at system level or on any resource, either by 
  name or with the operation's OperationDefinition (`ValueSet-expand`, `ConceptMap-closure`...)
* `validate`: the server declares `$validate-code`, on CodeSystem or ValueSet (or at system level)
* `tx-resource`: the server declares the feature 
  `http://hl7.org/fhir/uv/tx-ecosystem/FeatureDefinition/CodeSystemAsParameter` with the value 
  `true` (see [Requirements](requirements.html)), so it accepts CodeSystem resources in the 
  `tx-resource` parameter

The `closure` suite uses this: it runs on servers that declare `$closure`. Its tests build a 
closure table on the server, with a fixed name, and start by resetting it with `reset = true`. 
A feature name the runner doesn't know (a typo, say) is never declared, so whatever needs it 
doesn't run, and the runner warns about it.
