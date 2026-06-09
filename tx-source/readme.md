= Introduction =

This folder contains testing material for large external code systems;
these are the sources for the test cases on LOINC, SCT etc.

What follows is notes about how to prepare the subsets

== SNOMED CT ==

The data under the `snomed/` folder is a subset extracted from the
`SnomedCT_InternationalRF2_PRODUCTION_20250801T120000Z.zip` release.

To regenerate the subset, take that international SNOMED distribution and run:

```
java -jar --add-opens java.base/java.lang=ALL-UNNAMED snomed-owl-toolkit-5.3.0-executable.jar \
 -rf2-to-owl -rf2-snapshot-archives {zip-download} -version 20250801
java -jar --add-opens java.base/java.lang=ALL-UNNAMED snomed-subontology-extraction-2.2.0-SNAPSHOT-executable \
 -source-ontology ontology-20250801.owl -input-subset {snomed-subset.txt} -output-rf2 -rf2-snapshot-archive {zip-download} -include-inactive
```
where `{snomed-subset.txt}` is the file in this folder.

=== Import ===

After importing the snapshot files, you must create a **beta version**
with effective time `20250814` so that the version URI returned by the server
matches what the tests expect. e.g. Mark it as an **internal release** in Snowstorm —
this causes the server to use `xsct` (not `sct`) in the version URI:
```
http://snomed.info/xsct/900000000000207008/version/20250814
```

Once the server is running, run the `snomed-expand-count-all` test as a quick
regression check to verify the expected total concept count is returned.

== LOINC CT ==

The subset is prepared by the code at https://github.com/HealthIntersections/nodeserver,
and running

```
npm install
tx-import loinc-subset
```

You then have to provide a reference to a downloaded LOINC source, 
and a list of LOINC codes to import, which is found in
loinc-subset.txt in this folder

== NDC ==

The test set is prepared by hand by taking subsets
from the first few lines of NDC

