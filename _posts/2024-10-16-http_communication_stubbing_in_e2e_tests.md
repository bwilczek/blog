---
layout: post
title:  Stubbing HTTP communication in E2E test with Hoverfly
date:   2024-10-16 09:05:15 +0200
excerpt: Building a robust solution for faking HTTP communication in distributed E2E testing stack
categories:
---

Stubbing out communication with external services is a common practice in automated testing. It brings various advantages, for example:

* reduced flakiness
* faster execution
* ability to test edge cases (network errors)

It is pretty easy to stub requests to remote services in unit tests, where the test runner process is executing the code under test.
There's a variety of tools that hook into the HTTP client libraries and alter their behavior in runtime, making them an important tool
in the tester's tool belt. Ruby, Java, Python, Node, and other have their `VCR`, `sinon`, `puffing-billy`, `Betamax`, `Talkback`.
None of these tools however has been designed to provide a technology agnostic solution for complete distributed app stack.

For E2E testing communication stubbing is not simple because the test runner lives in a different process
then the application under test. And this makes changing the behavior of the running app in runtime impossible.
Or at least very, very hacky and standing in the way of E2E testing paradigm.

Additionally the application under test can be multi-process itself: consider clustered HTTP server and a separate service
for processing of background jobs. In order to provide consistent behavior of the whole stack all processes should be experiencing
the same responses for the same requests. Tools listed in previous paragraph won't help in this case.

## Hoverfly enters the stage

The solution to this problem is making all the processes in the stack route their HTTP communication through a proxy server
and let this proxy perform any manipulation on the responses, as required by the test scenario.

[Hoverfly](https://docs.hoverfly.io/en/latest/) is (almost) ideal tool for this purpose.
It hooks itself into the HTTP requests processing seamlessly, without needing to alter the applications under test.

## Settings things up

All applications involved in E2E test scenarios have to be configured to use an HTTP proxy server for external request they make.
Setting this up can vary, depending on the libraries used.

# Starting Hoverfly

But first things first. Let's start a local instance of Hoverfly, that the other services will use:

```sh
docker run --name hoverfly -d -p 8888:8888 -p 8500:8500 spectolabs/hoverfly:latest
```

It is important to know where services under test run. If they run in docker containers Hoverfly container should be
created in the same network, and referred by name `hoverfly`. If the services run on the host (locally) the correct address
for Hoverfly service would be `localhost` or `127.0.0.1`.

As one can see two ports are being exposed: `8500` for the actual proxy and `8888` for the admin interface and the REST API.

# Configuring proxy with env variables

In ideal scenario the desired configuration should be achieved without needing to change a single line of application's code,
using only environment variables. There's an (informal) standard for proxy configuration, that involves three variables:

* `http_proxy` - holds proxy URL to be used for HTTP requests
* `https_proxy` - holds proxy URL to be used for HTTPS requests
* `no_proxy` - holds a coma separated list of URLs that should not go through a proxy. Typically `localhost` or other services from the same app stack.

As stated before our instance of Hoverfly runs at `http://hoverfly:8500`.
Then a typical set of variables for E2E test stack would be:

```shell
export http_proxy=http://hoverfly:8500
export https_proxy=http://hoverfly:8500
export no_proxy=localhost,127.0.0.1,hoverfly,db,web,search,other,services
```

Most likely these would be declared in GitHub Action workflow, or `docker-compose.yml`, or `kubernetes` manifests, or wherever the stack is started.

# Configuring proxy per library

Some HTTP client libraries do not respect these env vars. If this is the case the configuration will have to be provided in the code.
For example `node-fetch` requires some tweaking:

```javascript
const fetch = require('node-fetch');
const HttpsProxyAgent = require('https-proxy-agent');

const agent = new HttpsProxyAgent(process.env.HTTPS_PROXY);

fetch('https://example.com', { agent })
  .etc()
```

Fortunately, the changes are minimal and (assuming that apps under test are properly designed) should happen in just one place in the code.

# Configuring SSL certificate

In order to support requests to HTTPS sites Hoverfly comes with a bundled [SSL certificate](https://github.com/SpectoLabs/hoverfly/blob/master/core/cert.pem).
However, as this is a testing tool, this certificate is self-signed, and needs to be marked as "trusted" before the applications
using the proxy will attempt to connect to secure sites.

Similarly as with proxy URL this goal can be achieved in a way transparent to the apps' code, by trusting the certificate on a global (OS) level.
On `debian` systems it can be done with:

```sh
cp cert.pem /usr/local/share/ca-certificates/hoverfly.crt && update-ca-certificates
```

Not all apps however respect this approach. If this is the case with your HTTP client please refer to its documentation to find a proper solution.
It shouldn't be too hard though, as often the issue can be resolved with an environment variable. For example:

* Python's `requests` package: `REQUESTS_CA_BUNDLE=cert.pem`
* Node's extra CA setting: `NODE_EXTRA_CA_CERTS=cert.pem`
* Curl's extra CA setting: `CURL_CA_BUNDLE=cert.pem` (it should respect system settings though)

## Example workflow

Once the applications are configured to use Hoverfly proxy the actual testing workflow can start.
Before the tests can be executed the responses have to be prepared.
In Hoverfly terms a collection of request/response pairs is called a [Simulation](https://docs.hoverfly.io/en/latest/pages/keyconcepts/simulations/simulations.html).

There are two ways of creating simulations.

# Recording a simulation

Hoverfly can work in several [modes](https://docs.hoverfly.io/en/latest/pages/keyconcepts/modes/modes.html).
For this article we care only about `capture` and `simulate` modes. First for recording real traffic,
second for serving the pre-recorder responses, not allowing any connections to real systems.

The are several ways to set the mode to the required value. First is using the admin panel at `http://localhost:8888`.

Another way is using CURL and REST API:

```sh
curl -X PUT -H "Content-Type: application/json" -d '{"mode": "capture"}' http://localhost:8888/api/v2/hoverfly/mode
```

The last way is calling the REST API from the code. The code examples in this article use `TypeScript` and [@bwilczek/hoverfly-client](https://www.npmjs.com/package/@bwilczek/hoverfly-client) package.

```typescript
import { Client } from "@bwilczek/hoverfly-client"

const client = new Client("http://localhost:8888")
await client.setMode({mode: 'capture'})
```

After the mode has been set Hoverfly is ready to record the traffic going through it.
It's time to let the apps under test perform their requests and let Hoverfly record them.
Typically app user (or acceptance tester, so most like you, dear reader) will just one the app UI and click through the functionality in question.

When the scenario is completed it's time to save the recorded traffic to a file, so that it could be reused in the future.
This can be achieved it two ways.

Using CURL and REST API:
```sh
curl http://localhost:8888/api/v2/simulation > simulation.json
```

Or from `TypeScript` code:
```typescript
const sim = await client.getSimulation()
saveSimulationToFile(sim, 'simulation.json')
```

The saved `simulation.json` file can now be edited in any text editor to remove any redundant data, simplify response body,
or provide responses for specific edge case scenarios.

# Crafting a simulation in the code

Another approach is crafting the simulation programmatically, in the code. Here's an example:

```typescript
import { 
  buildSimulation,
  ResponseData,
  RequestMatcher,
  saveSimulationToFile
} from "@bwilczek/hoverfly-client"

const response: ResponseData = {
  status: 402,
  body: '{"result": "error", "message": "Insufficient balance"}',
  encodedBody: false,
  templated: false
}

const request: RequestMatcher = {
  path: [{ matcher: 'exact', value: '/api/invoices' }],
  destination: [{ matcher: 'exact', value: 'payment.provider' }],
}

const pair = { request: request, response: response }

const sim = buildSimulation([pair])
saveSimulationToFile(sim, 'simulation.json')
```

# Serving responses from a simulation

Once the traffic is recorded in a simulation file it's time to change Hoverfly mode to `simulate`
and start using the pre-recorded responses instead of real services.

Changing mode is easy:
```
# curl:
curl -X PUT -H "Content-Type: application/json" -d '{"mode": "capture"}' http://localhost:8888/api/v2/hoverfly/mode

# TypeScript:
await client.setMode({mode: 'simulate'})
```

Uploading the simulation JSON isn't hard as well:

```
# curl:
curl -X PUT -H "Content-Type: application/json" -d @simulation.json http://localhost:8888/api/v2/simulation

# TypeScript:
await client.uploadSimulation(buildSimulationFromFile('simulation.json'))
```

Obviously simulations crafted by hand can be uploaded without having to be saved to a JSON file first.
```typescript
const sim = buildSimulation([pair])
await client.uploadSimulation(sim)
```

# Middleware and Journal

Hoverfly comes with two more handy tools that can provide more flexibility to the test framework.
First one is [middleware](https://docs.hoverfly.io/en/latest/pages/keyconcepts/middleware.html),
a mechanism that can modify the responses dynamically, make requests to real systems, trigger callback webhooks
or perform any logic that could be implemented inside a simple HTTP application.
This topic is so broad and project specific that it goes far beyond the scope of this article.
It's being mentioned here so that you know that if there's something more sophisticated that your
stubbed communications needs to do, middleware can help.

The other concept is Journal, which is basically a registry of performed HTTP requests that went through Hoverfly.
It's essential for making assertions like *"Expect that payment provider has been queried for pending transactions"*.
More on this in the example below.

## Example workflow with hoverfly-client

Let's demonstrate the features of Hoverfly in an example `jest` test:

```typescript
import { describe, expect, test } from '@jest/globals'
import {
  buildSimulation,
  ResponseData,
  RequestMatcher,
  saveSimulationToFile,
  Client
} from "@bwilczek/hoverfly-client"

const client = new Client("http://hoverfly:8888")

describe("Fetch invoice", () => {
  beforeEach(async () => {
    await client.purgeSimulation()
    await client.setMode({mode: 'simulate'})
    await client.purgeJournal()
    // upload some default requests/responses that should be always active, for example authenthication
    await client.uploadSimulation(buildSimulationFromFile('default_traffic.json'))
  })

  test('sufficient balance', async () => {
    // append simulation for this scenario to the one already present in Hoverfly
    await client.appendSimulation(buildSimulationFromFile('payment_sufficient_balance.json'))

    // do some actions in the UI, that will make the app under test perform a request to payment provider
    await browser.submitPaymentButton.click()

    // assert that the backend really performed a request to the payment provider
    const paymentsJournal = await client.searchJournal({request: {destination: [{matcher: "exact", value: "payment.provider"}]}})
    expect(paymentsJournal.journal.length).toBe(1)
  })
})
```

## Conclusions

Stubbing HTTP communication in a distributed, multi-process system is not only possible, but it's also not that hard.
HTTP proxy is the right tool for this purpose, and Hoverfly provides just the right features:

* Flexible request matchers
* Middleware for custom logic
* Support for SSL
* Journal for tracking of the processed requests
* JSON REST API for easy configuration
* Simulating latency and outages

As demonstrated in this article, introduction of Hoverfly to any app stack is easy.
With minimal or no changes to the code it opens the apps for testing of
use cases not achievable with requests to real external services.

## Caveats

As great as Hoverfly is it comes with a few caveats:

**`destination`, `no_proxy` and direct requests**

Switching Hoverfly on and off dynamically for certain URLs is hard. `no_proxy` variable provides a static list,
while [destination](https://docs.hoverfly.io/en/latest/pages/keyconcepts/destinationfiltering.html)
filtering relies on a whitelist regular expression.
Implementation of a test suite that runs some scenarios against real payment provider and some against a simulated one
is tricky and requires some fancy logic in a stateful middleware. It's doable though.

**not every lib respects `http_proxy`**

As described above: some HTTP libraries won't respect the standard ENV variables
and require changes to client initialization.

**not every lib respects default location of SSL certificates**

As described above: some HTTP libraries won't trust all system's certificates
and require changes to client initialization.

**default certificate does not work well with time traveling**

Scenarios that involve time traveling might not work as expected, as the SSL certificate shipped 
with Hoverfly has more "present" validity period. A custom certificate, valid +/- 30 years from now
could be generated and used instead.

**one, huge simulation, cannot upload/delete single pairs**

REST API operates (upload/download) on a whole simulation - not individual request/response pairs.
Adding or subtracting responses requires some extra coding, fortunately it can be abstracted away in a
client library, like the one used in the examples in this article.

**JSON format of simulations**

Pretty often request or response payload is also in JSON format, what makes manual changes to the simulation files hard.
JSON does not support line splitting and the JSON content is at easiest case escaped and 
stored in a very long line. In a harder case it can be `gzip` or `brotli` compressed and then serialized in `base64`.
Of course editing such payload is still possible, but a bit harder then if a different format was used.
