# alien

###### Automated Load Inducing Emulation Network

Alien was developed to overcome a limitation of `ab(1)` (Apache Benchmark) so that each request is generated so that they can be different.

The custom modules give `alien` a lot of power. It is possible to mock a whole client-server conversation with a server, and then run those conversations concurrently.

## Build Instructions

Alien is designed to be a standalone executable with versions for linux, windows and macos though it can be run directly by `node`.

The `alien` executable is the nodejs runtime bundled with the javascript code and generated by [pkg](https://www.npmjs.com/package/pkg).

    git clone https://github.com/redskyit/alien
    cd alien
    npm i
    npm run build

On successful build the `dist` folder contains the built executables as well as a minified and uglified `alien.js` for use with `node`.

## Usage

```
alien -r [concurrent-tests] -n [request-count] -c [concurrent-requests] -t [test-module.js] --fail-on-text [text] --fail-on-expr [regex] [test params]
```

The `-r` option specifies the number of tests that will be run. By default only one test is run. A test is defined as a series of requests defined by the test module. Separate instances of the test are run concurrently.

The `-n` option specifies the number or requests a test should make. Requests are run sequentially unless the `-c` option is also specified.

The `-c` option specifies the number of requests that should be run concurrently. When specified, requests are bundled into batches of concurrent requests. Batches are run sequentially. So for example, `-n 10 -r 5` will run two sequential batches of 5 concurrent requests.

The `-t` option specifies the test module to use. The default test module (built into alien) allows for a simple GET request to be repeated. A custom test module allows more control over the types of requests being made.

The `--fail-on-text` option will cause a response from the server to be treated as a failure, if it succeeds but the body of the response contains the string specified by this option.

The `--fail-on-expr` option will cause a response from the server to be treated as a failure, if it succeeds but the body of the response matches the regular expression specified by this option.

## Default Module

The default module supports simple HTTP requests. Given just a URL, a GET request is made. To allow arguments to be passed to the default module, add -- before the URL argument. For example:

    alien -n 1 -- http://localhost/ -m <http-method> -b <body> -f <body-file> -e <body-file-encoding> -T <content-type> -H <header-list>

The `-m` option specifies the HTTP method to use, GET, PUT, POST, DELETE etc.

The `-b` option specifies the body of the request as a string.

The `-f` option loads the body of the request from the specified file.

The `-e` option specifies the encoding used by the file specified width the `-f` option. For instance, if the file contains UTF8 encoded text, specify `-e utf8` to treat the body as text content, otherwise it will be treated as a binary stream.

The `-T` option sets the `Content-Type` header.

The `-H` option is a `|` (pipe) separated list of headers, with each header being being `<Header>: <value>` for example

    -H 'Cache-Control: no-cache|Content-Type: text/xml; charset=UTF-8'

## Test Modules

Test modules are javascript files provided by the user that implement the test module interface. Their task is to feed alien with request details, and to report on results.

### Test module interface

The test module interface consists of a number of exported async methods, all but one are optional, and an API provided by Alien to the test module.

A single test module is used for all concurrent tests. The module must take steps to keep individual test parameters separate from other tests. Alien facilitates this by providing a state object to each test for it to place its test state in.

#### Test Module Lifecycle Methods

```javascript
async function onload({ alien })
  async function startup({ alien })
    async function next({ alien, ... })
    ...
  async function shutdown({ alien })
  async function report({ alien, ... })
async function onunload({ alien })
```

***`async function onload({ alien })`***

`onload` is called before any tests start. It can be used for parsing test arguments `alien.args` and any other pre-test initialisation.

***`async function startup({ alien })`***

`startup` is called for a test instance before the test starts, it can be used to initialise the test instance (run) state `alien.run.state` which is an object provided by alien that is private to this test instance.

***`async function next({ alien, ts, state, results, batchResults, batchIndex, requestIndex })`***

`next` is called each time `alien` starts a new request. It is provided details about the current request, as well as given access to the results of the previous batch of requests.

| Argument | Description
|------:|----------------------------------------------------
| `ts` | A time stamp in the format YYYYMMDD-HHMMSS-CCC
| `state` | Test instance state
| `results` | Results (so far)
| `batchResults` | Results from the last batch of concurrent requests
| `batchIndex` | The current batch index.
| `requestIndex` | The current request index.

The next method returns details of the next request as an object with the following structure:

```typescript
{
  method: String,
  url: String,
  body: String,
  cookies: Object<String,String>,
  headers: Object<String,String>
}
```

| Property | Description
|---------:|----
| `method` | The request method, GET, PUT, POST, DELETE etc. Optional, the default is `GET`
| `url` | The request url, including any query string
| `body` | The request body. Optional.
| `headers` | Request headers as key-value pairs: 
|| `{ 'Content-Type': 'text/xml', ... }`
| `cookies` | Cookies to send to the server:
|| `{ 'User-Agent': 'example-test-module', ... }`

***`async function shutdown({ alien, results })`***

`shutdown` is called after the last request for a test instance has completed.

***`async function report({ alien, results, summary })`***

`report` is called after `shutdown` and is given the test instance results along with some statistics about that test.  The test module can generate its own statistics from the tests results provided.

***`async function onunload({ alien })`***

`onunload` is called after the last test has completed.

## Examples

##### Basic
`alien -n 100 http://localhost/` 
##### Basic Concurrent
`alien -n 100 -c 10 http://localhost/`
##### Module 
`alien -n 100 -t example.js http://localhost/ 123 TEST`

## Alien API

The Alien API is an interface provided to a test module to provide information or assist it in generating requests or stats.

##### `alien.lpad()`

    alien.lpad(string, width, z = ' ')

Left pads a string, or number to a specified with, and optionally allows the pad character to be specified, with space being the default. Returns the padded string.

##### `alien.showSummary()`

    alien.showSummary(summary, { percents: Boolean })

Outputs a summary of the test, and optionally include the table of request percentages vs response time ranges.

##### `alien.showResponseTimePercentages()`

    alien.showResponseTimePercentages(summary.percents);

Outputs the table of request percentages vs response time ranges.

##### `alien.handleASPNETSessionCookie()`

    alien.handleASPNETSessionCookie(alien.run);
    return { url: "...", cookies: alien.run.state.cookies, headers: [] };

Picks up any `ASP.NET_SessionId` cookie sent by the server, and updates `alien.run.state.cookies` to include that cookie, which can then be sent back to the server so that subsequent requests run in the same session.

##### `alien.showFailedRequests()`

    alien.showFailedRequests(results)

Dumps details of each failed request from the request results. It tries to extract error information from the response text and display that, or failing that will dump the whole response text.

##### `alien.checkMinVersion()`

    alien.checkMinVersion('1.0.6')

Returns true if the version of alien is equal to or greater than the specified version.

##### `alien.args`

The command line arguments (excluding alien defined arguments). For the default test, these are ignored. For a custom test, this includes all the arguments passed on the command line following the standard alien options.  For example:

    alien -n 20 -t timesheet.js <service-uri> <device-id> <project>

A custom test can use these arguments as input to their tests, in the above example, providing details required to run the test.s

##### `alien.env`

A hash containing current environment variables. Same as process.env.  Environment variables can be references as follows (both examples yield the same result)

    alien.env.TESTROOT
    alien.env['TESTROOT']

##### `alien.tests`

Details about the tests being run.

##### `alien.tests.concurrent`

The number of concurrent tests that are being run. From the command line option `-r <n>`

##### `alien.shared`

An object that is shared across all concurrent tests. Could be used for sharing information between concurrent tests.

##### `alien.run`

Each concurrent test has its own run environment, alien.run is that environment.

##### `alien.run.test`

The test ID. Assigned sequentially one for each concurrent test.

##### `alien.run.requests`

The number of requests to make. From the `-n <n>` command line argument.

##### `alien.run.concurrent`

The number of requests to run concurrently. From the `-c <n>` command line argument.

##### `alien.run.results`

An array containing the results of each request as it completes. The order of this array is unpredictable. If the order needs to be predictable the test module must sort the results by requestIndex.

##### `alien.run.start`

Timestamp (in milliseconds) when this test started.

##### `alien.run.cookies`

As requests complete that include the `Set-Cookie` header, these are stored here. Cookies are reflected back to the server in each subsequent request.

##### `alien.run.state`

An object that is shared across requests, but not across concurrent tests. Alien does not rely on the contents of state so the module can store anything it wants within this object.
