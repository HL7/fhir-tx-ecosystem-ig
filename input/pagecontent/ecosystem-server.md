The Coordination server keeps a current set of information about the servers in its
ecosystem, and users of the ecosystem ask the Coordination server which terminology server should be
used for a particular operation. 

#### Scanning the Ecosystem

The Coordination server scans the servers referenced from the master registration 
file on a regular basis, and maintains a live list of the servers, their configuration, and what 
CodeSystems and ValueSet each endpoint supports and is authoritative for. The frequency of scanning 
is a matter for the ecosystem and server administrators, but is generally every few minutes.

The Coordination server will use the ```/metadata``` and ```/metadata?mode=terminology```
operations, and also iterate the full set of ValueSets on the server using ```/ValueSet?_elements=url,version&_count=1000```,
following the `next` links. A server that doesn't support `_elements`, or that returns smaller pages, 
simply returns more content or more pages.

Note that the coordination server does not search the CodeSystems; instead, the list of 
supported CodeSystems comes from the TerminologyCapabilities resource.

### Authentication/Security 

If the server requires authentication for these operations, the Coordination server
must be given access to these operations. 

