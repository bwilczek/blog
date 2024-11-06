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
None of these tools hover is design to provide a technology agnostic solution for a whole distributed app stack.

For E2E testing communication stubbing is not simple because the test runner occupies a different process
then the application under test. And this makes changing the behavior of the running app (HTTP client libs) in runtime impossible.
Or at least very very hacky and standing in the way of E2E testing paradigm.

Additionally the application under test can be multi-process itself: consider clustered HTTP server and a separate service
for processing of background jobs. In order to provide consistent behavior of the whole stack all processes should be experiencing
the same responses for the same requests. Tools listed in previous paragraph won't help in this case.

## Hoverfly enters the stage

The solution to this problem is making all the processes in the stack route their HTTP communication through a proxy server
and let this proxy perform any manipulation on the responses, as required by the test scenario.

[Hoverfly](https://docs.hoverfly.io/en/latest/) is (almost) ideal tool for this purpose.
It hooks into the HTTP requests processing seamlessly, without needing to alter the applications under test.

## Settings things up

All applications involved in E2E test scenarios have to be configured to use an HTTP proxy server for external request they make.
Setting this up can vary, depending on the libraries used.

# Starting hoverfly

But first things. Let's start a local instance of Hoverfly, that the other services will use:

```sh
docker run --name hoverfly -d -p 8888:8888 -p 8500:8500 spectolabs/hoverfly:latest
```

It is important to know realize where services under test run. If they run in docker containers Hoverfly container should be
created in the same network, and refered by name `hoverfly`. If the services run on the host (locally) the correct address
for Hoverfly service would be `localhost` or `127.0.0.1`.

As one can see two ports are being exposed: `8500` for the actual proxy and `8888` for admin interface and REST API.

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

Most likely these would be declared in GitHub Action workflow, or `docker-compose.yml`, or `kubernetes` manifests, or whenever the stack is started.

# Configuring proxy per library

Some HTTP client libraries do not respect these env vars. If this is the case the configration will have to be provided in the code.
For example `node-fetch` requires some tweaking:

```javacript
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
Before the tests can be executed the forged responses have to be prepared.
In Hoverfly terms a collection of request/response pairs is called a [Simulation](https://docs.hoverfly.io/en/latest/pages/keyconcepts/simulations/simulations.html).

There are two ways of creating simulations.

# Recording a simulation

Hoverfly can work in several [modes](https://docs.hoverfly.io/en/latest/pages/keyconcepts/modes/modes.html).
For this artical we care only about `capture` and `simulate` modes. First for recording real traffic,
second for serving the pre-recorder responses, not allowing any connections to real systems.

The are several ways to set the mode to the required value. First is using the admin panel at http://localhost:8888.

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
It's time to let the apps under test perform their requests and be recorded.
Typically app user (or acceptance tester, so most like you, dear reader) will just click through the functionality in question.

When the scenario is completed it's time to save the recorded traffic to a file, so that it could be reused in the future.
This can be achived it two ways.

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

Uplading the simulation JSON isn't hard as well:

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

# Journal

asd

# Middleware

asd

## Example workflow with hoverfly-client


asd

## Conclusions

asd

## Oh, but why almost ideal?

As great as `hoverfly` is it comes with some caveats:

* `destination`, `no_proxy` and direct requests
* not every app respects `http_proxy`
* not every app respects default location of SSL certificates
* default certificate does not work well with time traveling
* one, huge simulation, cannot upload/delete single pairs
* JSON format holding JSON responses as a single line, often encoded. Hard to tamper with.
