### Test cases 

The tests assume that the server can accept code systems on the fly. 
If servers do not accept code systems on the fly, server authors will have to 
consult the FHIR product director. Either way, servers that do SHOULD pass all the tests, but the FHIR product director 
will review the test outcomes in order to approve a server. 

The test cases are in version R5, but the tests will run against either an R4 or an R5 server. 
See [R4 and the Test Cases](r4.html)

Note that the test cases will be migrated to use TestPlan+matchetypes at some stage.

#### Running the tests 

The tests can be run by any runner that processes the tests correctly, but the easiest way to 
execute the tests is to use the [standard Java FHIR Validator](https://github.com/hapifhir/org.hl7.fhir.core/releases).
You do not download the tests yourself - the runner fetches the published test package for you:

````
java -jar validator_cli.jar txTests -tx {server} [-test-version {version}] [-output {folder}] 
    [-externals {file}] [-mode {mode},{mode}] [-suite {suite}] [-filter {text}] [-input {folder}]
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

Note that the FHIR version to test is *not* a parameter: the runner asks the server which version 
of FHIR it speaks, and runs the R4 or the R5 form of each test accordingly. 
See [R4 and the Test Cases](r4.html).

#### Test Output

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
output directory for each variant, or, in server mode, the `label` parameter described below.

#### Running the tests in a pipeline 

The simplest way to run the tests in a build is the command line above: it exits with 0 if every
test passed and 1 if any test failed, so a CI step needs no output parsing. Always pass
`-test-version` in a pipeline. The default is `current`, which is the ci-build of this IG and
changes whenever its master branch changes - a build that uses it can go red because the tests
moved, not because your server did. Pin a released version, and change it deliberately. Pass
`-output` as well, and keep the directory as a build artefact: when the step fails, the `expected/`
and `actual/` pair for each failing test is the only thing that tells you what went wrong, and it
is gone with the build container otherwise.

The drawback of the command line is that the whole run is a single result. If you want your own
test framework to report one result per test - to see which tests are newly failing, to track them
over time, or just to get a useful failure message next to the test that produced it - use the
validator's server mode instead. Start the validator once as an HTTP server, and ask it to run
one test at a time:

````
java -jar validator_cli.jar server {port} -version 5.0

GET http://localhost:{port}/txTest?server={url}&suite={suite}&test={test}&modes={modes}&folder={folder}&label={label}
````

Each call runs one test and returns an `OperationOutcome`. If it has no issue with a severity of
`error`, the test passed; otherwise `details.text` is the failure message - the same message the
command line would have logged. The test cases and the server's setup resources are loaded once,
on the first call for a given server, and reused, so the cost of starting a JVM and loading the
test package is paid once, not once per test. Drive it from whatever test framework you already
use: enumerate the suites and tests from `test-cases.json` in this IG's package, and make one call
per test.

Two of the parameters matter more in server mode than they look. `modes` replaces the runner's
mode set for that call: pass every mode your server supports, exactly as you would on the command
line, because a test gated on a mode you did not ask for does not run, and a test that did not run
is reported as neither a pass nor a failure but as an explanation of which mode would have run it.
Sending no `modes` at all gets a default set that may not match your server. `folder` and `label`
(validator 6.10.5 and later) control where the output goes: `folder` names the run's folder inside
the validator's temporary directory - a simple name, not a path, defaulting to the server's host
name - and `label` names a subfolder of it for that one test. Give each variant of a run its own
`label` (`r4`, `r5`, `r5-cached`) and the diffs for a test that fails in only one of them survive.

#### Versions of the test cases

The test cases change: tests are added, and corrected, as servers and the specification develop. 
So this IG is released periodically, and each release is a fixed set of test cases that you can 
test against repeatedly and compare results over time. The releases are listed on the 
[history page](history.html), and you choose one with `-test-version`:

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

#### Test Suites and Test Cases 

All the tests are registered in `tests/test-cases.json`, which contains a list of suites, 
each of which contains a list of tests. A suite may have:

* `name`: the name of the suite (used in the test output, and to run a single suite)
* `description`: what the suite is for
* `setup`: a list of files containing the resources (code systems, value sets, concept maps) 
  that the server needs in order to run the tests in the suite. Servers that cannot accept 
  resources on the fly have to load these some other way
* `mode` / `modes`: the suite only runs when the runner is in that mode (see below)
* `version`: the suite only runs against servers of that FHIR version (see below)
* `disabled`: if true, the suite is not run at all

A test may have:

* `name`: the name of the test, unique within the suite
* `description`: what the test is checking. This is the documentation for the requirement 
  the test enforces, so it should say why the expected response is the correct one
* `operation`: one of `expand`, `validate-code`, `cs-validate-code`, `lookup`, `translate`, 
  `compare`, `batch`, `batch-validate`, `metadata`, `term-caps`
* `request`: the file containing the request parameters (not used by `metadata` / `term-caps`)
* `response`: the file containing the expected response
* `request:{mode}` / `response:{mode}`: an alternative request or response used when the 
  runner is in that mode - e.g. `response:flat` for servers that return a flat expansion
* `mode` / `modes`, `version`, `disabled`: as for suites
* `http-code`: the expected HTTP status code, where it isn't 200 (e.g. `422`, or `4xx`)
* `lenient-display`: sets the `lenient-display-validation` parameter on a `validate-code` or 
  `cs-validate-code` request, so that the same request can be tested both ways
* `profile`: a file containing a Parameters resource with the expansion parameters to use
* `Accept-Language`: the language to ask for
* `header`: an additional HTTP header to send ( `name`, `value`, and optionally `mode`)

#### FHIR Versions 

The test cases are written in R5, and most of them are the same for an R4 server, but 
some questions have different answers in different versions of FHIR. Where a test only 
makes sense for particular versions, the `version` property says which:

* `"version" : "4.0"`: only run this against an R4 server (matches `4.0`, and any version 
  starting `4.0.`, so `4.0.1` matches)
* `"version" : "!4.0"`: run this against anything except an R4 server

Where the same test applies to all versions but the *response* differs, the difference is 
marked in the expected response with `$optional$` or `$only$` (see below).

#### Test Templates 

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

#### $optional$ and $only$ 

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

#### Modes 

Some tests, and some parts of expected responses, only apply to servers with particular 
characteristics, or to a particular server. These are marked with a mode, and only run 
when that mode is passed to the runner with the `-mode` parameter. `general` is always 
on unless it is turned off with `-mode !general`. The modes in use are:

* `general`: the tests every server is expected to pass. This is the default
* `snomed`: servers that support SNOMED CT, loaded with the test subontology described in 
  `tx-source/readme.md`
* `omop`: servers that support OMOP
* `icd-11`: servers that support ICD-11
* `mimetypes`: servers that support the mime types code system (BCP 13, `urn:ietf:bcp:13`)
* `tx.fhir.org`: tests that are specific to tx.fhir.org - either its own bugs, or operations 
  that are still being trialled there. No other server is expected to pass these
* `flat`: servers that return a flat expansion rather than a hierarchical one. This mode 
  selects the `response:flat` alternative for the tests that have one

A mode name can also appear in `$optional$` / `$only$` in an expected response, so that a 
particular server (or every server but one) is held to a different requirement than the rest 
- `"$optional$" : "!tx.fhir.org"` is used throughout for content that only tx.fhir.org is 
required to return.

#### Registry

{% json tests/test-cases.json liquid/test-cases.liquid %}
