<h5 align="center">
Run <a href="https://developers.google.com/web/tools/lighthouse">Lighthouse</a> and <a href="https://github.com/pa11y/pa11y">Pa11y</a> audits directly in <a href="https://cypress.io/">Cypress</a> test suites
</h5>

---

[![Build Status](https://travis-ci.org/mfrachet/cypress-audit.svg?branch=master)](https://travis-ci.org/mfrachet/cypress-audit) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

- [Why cypress-audit?](#why-cypress-audit)
- [Usage](#usage)
  - [Preparation](#preparation)
  - [In your code](#in-your-code)
  - [cy.pa11y()](#cypa11y)
  - [cy.lighthouse()](#cylighthouse)
    - [Good to know before](#good-to-know-before)
    - [Thresholds per tests](#thresholds-per-tests)
    - [Globally set thresholds](#globally-set-thresholds)
    - [Passing options and config to Lighthouse directly](#passing-options-and-config-to-lighthouse-directly)
  - [Accessing the raw reports](#accessing-the-raw-reports)

## Why cypress-audit?

We have the chance of being able to use powerful tools to automated and prevent from different kind of regressions:

- [Cypress](https://cypress.io/) has made business oriented automated verifications easy
- [Lighthouse](https://developers.google.com/web/tools/lighthouse) has provided tools and metrics concerning applications performances
- [Pa11y](https://pa11y.org/) has provided tools to analyze and improve the accessibility status of applications

While these tools are amazingly powerful and helpful, I'm always feeling in pain when I try to use all of them in my projects.

For example, how can I verify the performance and accessibility status of a page requiring authentication? I have to tweak Lighthouse and Pa11y configurations (that are different) and adjust my workflows accordingly.

This is cumbersome because I already have my authentication logic and shortcuts managed by Cypress: why should I add more complexity in my tests?

The idea behind `cypress-audit` is to aggregate all the underlying configurations behind dedicated [Cypress custom commands](https://docs.cypress.io/api/cypress-api/custom-commands.html): you can benefit from your own custom commands and you can run cross-cutting verifications directly inside your tests.

## Usage

### Preparation

In order to make `cypress-audit` commands available in your project, **there are 3 steps to follow:**

#### Installing the dependency

In your favorite terminal:

```sh
$ yarn add -D cypress-audit
# or
$ npm install --save-dev cypress-audit
```

#### Preparing the server configuration

By default, if you try to run Lighthouse or Pa11y from the command line (or from Nodejs), you will see that they both open a new web browser window by default. As you may also know, Cypress also opens a dedicated browser to run its tests.

The following configuration allows Lighthouse, Pa11y and Cypress to make their verifications inside the same browser (controlled by Cypress) instead of opening a new one.

In the `cypress/plugins/index.js` file, make sure to have:

```javascript
const { lighthouse, pa11y, prepareAudit } = require("cypress-audit");

module.exports = (on, config) => {
  on("before:browser:launch", (browser = {}, launchOptions) => {
    prepareAudit(launchOptions);
  });

  on("task", {
    lighthouse: lighthouse(), // calling the function is important
    pa11y: pa11y(), // calling the function is important
  });
};
```

#### Making Cypress aware of the commands

When adding the following line in the `cypress/support/commands.js` file, you will be able to use `cy.lighthouse` and `cy.pa11y` inside your Cypress tests:

```javascript
import "cypress-audit/commands";
```

### In your code

After completing the [Preparation](#preparation) section, you can use the commands:

```javascript
it("should pass the audits", function () {
  cy.lighthouse();
  cy.pa11y();
});
```

### cy.pa11y()

![A Pa11y record showing some test failing on color contrast, landmark, heading and regions.](./docs/pally.png)

You can call `cy.pa11Y(opts)` with `opts` being any kind of [the pa11y options](https://github.com/pa11y/pa11y#configuration).

### cy.lighthouse()

![A Lighthouse record showing some test failing on best-practices and performances](./docs/lh.png)

#### Good to know before

Lighthouse is a tool that is supposed to run against a production bundle for computing the `performance` and `best-practices` metrics. But it's widely suggested by [Cypress to run their test on development environment](https://docs.cypress.io/guides/getting-started/testing-your-app.html#Step-1-Start-your-server). While this seems a bit counter intuitive, we can rely on the [Cypress project feature](https://docs.cypress.io/guides/guides/command-line.html#cypress-run-project-lt-project-path-gt) to run some dedicated test suites against production bundles and to have quick feedbacks (or prevent regression) on these metrics.

#### Thresholds per tests

If you don't provide any argument to the `cy.audit` command, the test will fail if at least one of your metrics is under `100`.

You can make assumptions on the different metrics by passing an object as argument to the `cy.audit` command:

```javascript
it("should verify the lighthouse scores with thresholds", function () {
  cy.lighthouse({
    performance: 85,
    accessibility: 100,
    "best-practices": 85,
    seo: 85,
    pwa: 100,
  });
});
```

If the Lighthouse analysis returns scores that are under the one set in arguments, the test will fail.

You can also make assumptions only on certain metrics. For example, the following test will **only** verify the "correctness" of the `performance` metric:

```javascript
it("should verify the lighthouse scores ONLY for performance and first contentful paint", function () {
  cy.lighthouse({
    performance: 85,
    "first-contentful-paint": 2000,
  });
});
```

This test will fail only when the `performance` metric provided by Lighthouse will be under `85`.

#### Globally set thresholds

While I would recommend to make per-test assumptions, it's possible to define general metrics inside the `cypress.json` file as following:

```json
{
  "lighthouse": {
    "performance": 85,
    "accessibility": 50,
    "best-practices": 85,
    "seo": 85,
    "pwa": 50
  }
}
```

_Note: These metrics are override by the per-tests one._

#### Passing options and config to Lighthouse directly

You can also pass any argument directly to the Lighthouse module using the second and third options of the command:

```js
const thresholds = {
  /* ... */
};

const lighthouseOptions = {
  /* ... your lighthouse options */
};

const lighthouseConfig = {
  /* ... your lighthouse configs */
};

cy.lighthouse(thresholds, lighthouseOptions, lighthouseConfig);
```

#### Available metrics

With Lighthouse 6, we're now able to make assumptions on **categories** and **audits**.

The categories are what we're used to with Lighthouse and provided a score between 0 and 100:

- performance
- accessibility
- best-practices
- seo
- pwa

The audits are things like the first meaningful paint and the score is provided in milliseconds:

- first-contentful-paint
- largest-contentful-paint
- first-meaningful-paint
- load-fast-enough-for-pwa
- speed-index
- estimated-input-latency
- max-potential-fid
- server-response-time
- first-cpu-idle
- interactive
- mainthread-work-breakdown
- bootup-time
- network-rtt
- network-server-latency
- metrics
- uses-long-cache-ttl
- total-byte-weight
- dom-size

### Accessing the raw reports

When using custom tools, it can be conveniant to directly access to raw information provided by the specific tool for doing manual things like generating a custom reports.

To do so, you can pass a `callback` function to the task initializer and when an audit is run, it will be triggered with the raw information.

In the `cypress/plugins/index.js` file:

```javascript
const { lighthouse, pa11y, prepareAudit } = require("cypress-audit");

module.exports = (on, config) => {
  on("before:browser:launch", (browser = {}, launchOptions) => {
    prepareAudit(launchOptions);
  });

  on("task", {
    lighthouse: lighthouse((lighthouseReport) => {
      console.log(lighthouseReport); // raw lighthouse report
    }),
    pa11y: pa11y((pa11yReport) => {
      console.log(pa11yReport); // raw pa11y report
    }),
  });
};
```
