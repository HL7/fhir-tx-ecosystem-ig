### Test cases 

The tests assume that the server can accept code systems on the fly.
If servers do not accept code systems on the fly, server authors will have to 
adapt these tests by rewriting them for their own actual support code systems. 
Either way, servers that do SHOULD pass all the tests, but the FHIR product director 
will review the test outcomes in order to approve a server. 

The test cases are in version R5, but the tests will run against either an R4 or an R5 server. 
See [R4 and the Test Cases](r4.html)

#### Running the tests 

The tests can be run by any runner that processes the tests correctly, but the  easiest way to 
execute the tests is to use the [standard Java FHIR Validator](https://github.com/hapifhir/org.hl7.fhir.core/releases).
Run with the parameters:

````
txTests -tx {server} -version? {version} -externals {file} -output {folder} -mode? {flat}?
````

Notes:
* server is the URL of the server to test for conformance 
* the version defaults to 'current' - the version of the tests in the ci-build of the this IG
* externals is a file that allows the server to use it's own messages - they don't have to match the tx.fhir.org messages. Just copy the file `tests/messages-tx.fhir.org.json` for the format.
* the output folder will default to your temporary directory. It produces output for the failed tests that you can compare with the /tests directory in the package for this IG with a comparison software of your choice (winmerge, beyondCompare, etc)
* modes - see below. you can pass in multiple modes if necessary, separated by commas 

#### Test Templates 

$$: any value

$xx$: value specifier: id, semver, url, token, string, date, version, uuid, instant, 

$optional-properties" : an array of string of properties that are allowed to be omitted

$optional$ : a boolean that can mark an object in an array as optional

$external:N$ : reference to string N in a list of external strings 

$count-array$ : ignore what's in the array, and just check the count 

$choice: array|$: a list of allowed values 

$fragments$: ??

#### Registry

{% json tests/test-cases.json liquid/test-cases.liquid %}