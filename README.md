## Go Package Management

Go takes a semi-decentralized approach to package management, allowing any git repo to be used as a module package system. Modules can be downloaded directly from their source control servers or alternatively through a module proxy which performs a simple forwarding of requests to the appropriate source. The following diagram is a high-level take on what happens when go get attempts to resolve a package or module.  


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c20xseigz99dca3uxr7n.png)

## Module Proxy

The module proxy is an HTTP server which implements the [module proxy protocol](https://go.dev/ref/mod#goproxy-protocol). It responds to GET requests matching a particular pattern, [See examples below](#go-module-proxy-request-patterns). The module proxy ensures changes or downtime to a module origin server will not bubble up and impact your build process. The module proxy can also act as a gatekeeper, restricting access to modules which have unsuitable licences or contain security vulnerabilities. Module proxies can also provide faster dependency resolution by caching previously fetched modules, which can then be used to serve future requests. A module's dependencies are defined in the following files:

| Module  | Description |
| ---| ----|
| `go.mod`| Defines the module name, the version of Go with which to build the project and the list of dependencies.|
| `go.sum`| Contains cryptographic hashes of the module's direct and indirect dependencies. |

### Module Proxy Protocol
Since version 1.13, [proxy.golang.org](proxy.golang.org) is the default module proxy for fetching modules.

It's possible to build your own module proxy, which consists of an HTTP server and REST API that satisfies the [GOPROXY protocol](https://go.dev/ref/mod#goproxy-protocol).

### Go Get Command
The `go get` command follows this protocol when attempting to resolve a package path:
- Search modules in the current project build list (go.mod) including all build lists of transitive dependencies, looking for modules with paths which are prefixes of the requested package path.
- If one module in the build list contains the required package, then that module will be used.
- If none of the modules in the searched build lists provide the package or if two or more modules provide the requested package, then `go get` will report an error.

When `go get` is resolving a new module for a package path , it uses the `GOPROXY` environment variable to determine whether modules will be downloaded directly, through a module proxy or some other private mirror. `GOPROXY` accepts a list of proxy URLs separated by either commas or pipes. The separator used in this list determines how the `go get` command will fallback in different scenarios:

|`GOPROXY` Separator Type | Behaviour |
| --- | ---|
| List separated by commas | `go get` will fallback to the next URL in the event of a 404 or 410 HTTP response. All other response codes are treated as terminal.|
| List separated by pipes | `go get` will fallback to the next URL in the proxy list in the event of any HTTP or non-HTTP error.|

The `go get` command visits each proxy in the list sequentially until it receives a successful response or error.

### GoProxy Environment Variable Options
The default configuration for  the `GOPROXY` environment variable is `GOPROXY=proxy.golang.org,direct` which effectively tells the `go get` command to first attempt module retrieval using the module mirror and if that approach is unsuccessful, fallback to the approach of connecting directly to the module origin repository. A number of other environment variables can also impact how `go get` will attempt to resolve dependencies. These variables are listed [here](#go-environment-variables-which-impact-module-retrieval).

| `GPROXY` keywords | Description |
|-------|------------------| 
| direct | `go get` will download directly from version control repo | 
| off | Disallows downloading from any source |

### Go Module Proxy Request Patterns
A GET request for a particular module will follow the pattern: `https://$base/$module/@v/$version.zip`. 

Where:

| Request portion | Description |Sample value |
|-------|------------------| ----------|
| $base | The path portion of the proxy URL | proxy.golang.org |
| $module | Module path |go.uber.org/zap |
| $version | Version | v1.21.0 |

The resulting GET request will download the module and dependencies from: `https://proxy.golang.org/go.uber.org/zap/@v/v1.21.0.zip`. The 'go get' and 'go mod tidy' commands abstract the details of this from the developer.

### Go Module Proxy Response Codes

The type of separator used in the `GOPROXY` environment variable will determine if the `go get` command will fall back to the next option in the list or fail completely. See section on ['go get'](#go-get-command).

| Proxy Response Code | Description |
|----|----|
| 200 | OK |
| 300 | Redirects are followed|
| 404 | Not found |
| 403 | Forbidden - Private proxies can choose to prevent access to modules based on unsuitable licenses or security vulnerabilities|
| 410 | Module not available on the proxy server but might be available elsewhere |
| 4xx & 5xx | Treated as errors |

### Go Environment Variables Which Impact Module Retrieval

If set with specific values, `GOPRIVATE` and `GONOPROXY` can impact the behaviour of `GOPROXY`, by preventing specific modules from being downloaded through the proxy.

| Environment Variable  | Description|
| ------ | ------ |
| `GOPRIVATE` | Comma separated list of module prefixes considered private. If set, this acts as a default value for `GONOSUMDB`|
| `GONOPROXY` | Comma separated list of module prefixes, which should always be downloaded directly from source. If set, this acts as a default value for `GOPRIVATE`  |
| `GONOSUMDB` | Comma separated list of module prefixes which the `go` command should not verify checksums using the checksum database.|

### Verifying Modules
After downloading a .mod or .zip file, `go get` computes a cryptographic hash and checks that it matches a hash in the main modules `go.sum`. If the hash is not present in `go.sum` then the go command retrieves it from the [checksum database](https://go.dev/ref/mod#checksum-database). The checksum database, located at `sum.golang.org`, allows for global consistency and reliability for all publicly available modules. It also ensures bits associated with specific module versions do not change from one day to the next. 

### Module Caching

The 'go get' command will cache modules downloaded from both module proxies and directly from a module's source repository in `$GOPATH/pkg/mod/cache/download`. This local cache layout is identical to the proxy URL space. 

### Module Graph 
Module graph pruning was introduced in version 1.17. See [go.dev](https://go.dev/ref/mod#graph-pruning) for more information. Module pruning uses [Minimal version selection](https://go.dev/ref/mod#minimal-version-selection) to select a set of module versions to use when building packages. It operates on a directed graph of modules, specified in go.mod. Each vertex in the graph represents a module version. Each edge represents a minimum required version of a dependency, specified using a require directive. The graph may be modified by exclude and replace directives in go.mod. MVS produces the build list as an output. 


| Command | Description |
|---| ----|
| `go list -m all` | Prints a modules build list |
| `go mod graph` | Prints the edges of the graph, one per line. Each line contains a module version and one of its dependencies  | 
| `go mod graph -go <some-version>`| Identical to previous command however it prints based on how the selected go version would interrupt the graph  | 
| `go mod verify`| Checks dependencies of the main module stored 
