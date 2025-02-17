---
title: Analyze results
excerpt: Use k6 to write custom metrics and filter results.
---

In this tutorial, learn how to:
- Apply tags to filter specific results
- Learn about k6 metrics
- Use [jq](https://jqlang.github.io/jq/) to filter JSON results
- Define groups to organize the test
- Create custom metrics


## Context: k6 result outputs


k6 provides many [result outputs](/results-output/overview/). 
By default, the [end-of-test summary](https://k6.io/docs/results-output/end-of-test/) provides the aggregated results of the test metrics.

<CodeGroup labels={["End of test summary (stdout)"]}  showCopyButton={[true]} heightTogglers={[true]}>

```bash
checks.........................: 50.00% ✓ 45       ✗ 45  
data_received..................: 1.3 MB 31 kB/s
data_sent......................: 81 kB  2.0 kB/s
group_duration.................: avg=6.45s    min=4.01s    med=6.78s    max=10.15s   p(90)=9.29s    p(95)=9.32s   
http_req_blocked...............: avg=57.62ms  min=7µs      med=12.25µs  max=1.35s    p(90)=209.41ms p(95)=763.61ms
http_req_connecting............: avg=20.51ms  min=0s       med=0s       max=1.1s     p(90)=100.76ms p(95)=173.41ms
http_req_duration..............: avg=144.56ms min=104.11ms med=110.47ms max=1.14s    p(90)=203.54ms p(95)=215.95ms
  { expected_response:true }...: avg=144.56ms min=104.11ms med=110.47ms max=1.14s    p(90)=203.54ms p(95)=215.95ms
http_req_failed................: 0.00%  ✓ 0        ✗ 180 
http_req_receiving.............: avg=663.96µs min=128.46µs med=759.82µs max=1.66ms   p(90)=1.3ms    p(95)=1.46ms  
http_req_sending...............: avg=88.01µs  min=43.07µs  med=78.03µs  max=318.81µs p(90)=133.15µs p(95)=158.3µs 
http_req_tls_handshaking.......: avg=29.25ms  min=0s       med=0s       max=458.71ms p(90)=108.31ms p(95)=222.46ms
http_req_waiting...............: avg=143.8ms  min=103.5ms  med=109.5ms  max=1.14s    p(90)=203.19ms p(95)=215.56ms
http_reqs......................: 180    4.36938/s
iteration_duration.............: avg=12.91s   min=12.53s   med=12.77s   max=14.35s   p(90)=13.36s   p(95)=13.37s  
iterations.....................: 45     1.092345/s
vus............................: 1      min=1      max=19
vus_max........................: 20     min=20     max=20
```

</CodeGroup>


For simplicity to learn about [k6 metric results](/using-k6/metrics/reference/), this tutorial uses the [JSON output](/results-output/real-time/json) and [jq](https://jqlang.github.io/jq/) to filter results.


For other options to analyze test results such as storage and time-series visualizations in real-time, review:
- [Results output](/results-output/overview/)
- [Ways to visualize k6 results](https://k6.io/blog/ways-to-visualize-k6-results/)



## Write time-series results to a JSON file

To output results as JSON lines, use the `--out` flag.

```bash
k6 run --out json=results.json api-test.js
```

Then run this `jq` command to filter the latency results; `http_req_duration` metric.

```bash
jq '. | select(.type == "Point" and .metric == "http_req_duration")' results.json
```

k6 results have a number of [built-in tags](/using-k6/tags-and-groups/#system-tags).
For example, filter results to only results where the status is 200:

```bash
jq '. | select(.type == "Point" and .data.tags.status == "200")' results.json
```

And calculate the aggregated value of any metric with any particular tags. For example, 

<CodeGroup labels={["Average", "Min", "Max"]} lineNumbers={[false]} showCopyButton={[true]} heightTogglers={[true]}>

```bash
jq '. | select(.type == "Point" and .metric == "http_req_duration") | .data.value' results.json | jq -s 'add/length'
```

```bash
jq '. | select(.type == "Point" and .metric == "http_req_duration") | .data.value' results.json | jq -s min
```

```bash
jq '. | select(.type == "Point" and .metric == "http_req_duration") | .data.value' results.json | jq -s max
```

</CodeGroup>

## Apply custom tags

You can also apply [_Tags_](/using-k6/tags-and-groups/#tags) to requests or code blocks.
To do so:

Add a [`tags`](/using-k6/tags-and-groups/#tags) object in the [request params](/javascript-api/k6-http/params/). Give the tag a key and value.

  ```javascript
  const params = {
    headers: {
      "Content-Type": "application/json",
    },
    tags: {
      "my-custom-tag": "auth-api",
    },
  };
  ```

Pass `params` to the `http` request. Create the file `tagged-login.js`:

  <CodeGroup labels={["tagged-login.js"]} showCopyButton={[true]} heightTogglers={[true]}>

   ```javascript
   import http from "k6/http";

   export default function () {
     const url = "https://test-api.k6.io";
     const payload = JSON.stringify({
       username: "test_case",
       password: "1234",
     });

     const params = {
       headers: {
         "Content-Type": "application/json",
       },
       //apply tags
       tags: {
        "my-custom-tag": "auth-api"
       },
     };

     //Login with tags
     http.post(`${url}/auth/basic/login`, payload, params);
   };
   ```
   </CodeGroup>

Run the test:

``` bash
k6 run --out json=results.json tagged-login.js
```

Filter the results for this custom tag:

```bash
jq '. | select(.type == "Point" and .metric == "http_req_duration" and .data.tags."my-custom-tag" == "auth-api")' results.json
```

## Organize requests in groups

You can also organize your test logic into [Groups](/using-k6/tags-and-groups#groups), test logic inside a `group` tags to all requests and metrics within its block.
Groups can help you to organize the test as a series of logical transactions or blocks.


### Context: a new test to group test logic

Results filtering isn't very meaningful in a test that makes one request.
And the API test script is getting long.
To learn more about how to compare results and other k6 APIs, write a test for the following situation:

> **A dummy example**: your development team wants to evaluate the performance of two user-facing flows.
> - visit an endpoint, then another one
>     - A GET request to `https://test.k6.io/contacts.php`
>     - A GET to `https://test.k6.io/`
> - play the coinflip game:
>     - A POST request to `https://test.k6.io/flip_coin.php` with the query param `bet=heads`
>     - Another POST to `https://test.k6.io/flip_coin.php` with the query param `bet=tails`


Can you figure out how to [script the requests](/using-k6/http-requests)?
If not, use the following script.

<Collapsible title="script example" isOpen="" tag="">

Since this example simulates a human user rather than an API call, it has a sleep between each request. Run with `k6 run multiple-flows.js`.

<CodeGroup labels={["multiple-flows.js"]} lineNumbers={["false"]} showCopyButton={[true]} heightTogglers={[true]}>

```javascript
import http from "k6/http";
import { check, group, sleep } from "k6";

//set URL as variable
const baseUrl = "https://test.k6.io";

export default function () {
  // visit contacts
  http.get(`${baseUrl}/contacts.php`);
  sleep(1);
  // return to the home page
  http.get(`${baseUrl}/`);
  sleep(1);

  //play coinflip game
  http.get(`${baseUrl}/flip_coin.php?bet=heads`);
  sleep(1);
  http.get(`${baseUrl}/flip_coin.php?bet=tails`);
  sleep(1);
}
```

</CodeGroup>

</Collapsible>

### Add Group functions

Wrap the two endpoints in different groups.
Name one group `Contacts flow` and another `Coinflip game`.

<CodeGroup labels={["multiple-flows.js"]} lineNumbers={["false"]} showCopyButton={[true]} heightTogglers={[true]}>

```javascript
//import necessary modules
import http from "k6/http";
import { check, group, sleep } from "k6";
import { Counter, Rate, Trend } from "k6/metrics";

//set baseURL
const baseUrl = "https://test.k6.io";

export default function () {

  // visit some endpoints in one group
  group("Contacts flow", function () {
    http.get(`${baseUrl}/contacts.php`);
    sleep(1);
    // return to the home page
    http.get(`${baseUrl}/`);
    sleep(1);
  });

  // Coinflip players in another group
  group("Coinflip game", function () {
    http.get(`${baseUrl}/flip_coin.php?bet=heads`);
    sleep(1);
    http.get(`${baseUrl}/flip_coin.php?bet=tails`);
    sleep(1);
  });
}
```

</CodeGroup>

### Run and filter

Inspect the results for only the `Coinflip game` group.
To do so:

1. Save the preceding script as `multiple-flows.js`.
1. Run the script with the command:

  ```bash
  k6 run multiple-flows.js --out json=results.json --iterations 10
  ```

1. Inspect the results with `jq`. Group names have a `::` prefix.

  ```bash
  jq '. | select(.data.tags.group == "::Coinflip game")' results.json
  ```

## Add a custom metric

As you have seen in the output, all k6 tests emit metrics.
However, if the built-in metrics aren't enough, you can [create custom metrics](/using-k6/metrics/create-custom-metrics).
A common use case is to collect metrics of a particular scope of your test. 

As an example, create a metric that collects latency results for each group:
1. Import [`Trend`](/javascript-api/k6-metrics/trend/) from the k6 metrics module.
1. Create two duration trend metric functions.
1. In each group, add the `duration` time to the trend for requests to `contacts` and the `coin_flip` endpoints.


<CodeGroup labels={["Adding custom metrics"]} lineNumbers={["false"]} showCopyButton={[true]} heightTogglers={[true]}>

```javascript
//import necessary modules
import http from "k6/http";
import {  group, sleep } from "k6";
import { Trend } from "k6/metrics";

//set baseURL
const baseUrl = "https://test.k6.io";

// Create custom trend metrics
const contactsLatency = new Trend("contacts duration");
const coinflipLatency = new Trend("coinflip duration");

export default function () {
  // Put visits to contact page in one group
  let res;
  group("Contacts flow", function () {
    // save response as variable
    res = http.get(`${baseUrl}/contacts.php`);
    // add duration property to metric
    contactsLatency.add(res.timings.duration);
    sleep(1);

    res = http.get(`${baseUrl}/`);
    // add duration property to metric
    contactsLatency.add(res.timings.duration);
    sleep(1);
  });

  // Coinflip players in another group

  group("Coinflip game", function () {
    // save response as variable
    let res = http.get(`${baseUrl}/flip_coin.php?bet=heads`);
    // add duration property to metric
    coinflipLatency.add(res.timings.duration);
    sleep(1);

    res = http.get(`${baseUrl}/flip_coin.php?bet=tails`);
    // add duration property to metric
    coinflipLatency.add(res.timings.duration);
    sleep(1);
  });
}
```

</CodeGroup>

Run the test with small number of iterations and output the results to `results.json`:

```bash
k6 run multiple-flows.js --out json=results.json --iterations 10
```

Look for the custom trend metrics in the end-of-test console summary:

```bash
coinflip duration..............: avg=119.6438  min=116.481  med=118.4755 max=135.498  p(90)=121.8459 p(95)=123.89565
contacts duration..............: avg=125.76985 min=116.973  med=120.6735 max=200.507  p(90)=127.9271 p(95)=153.87245
```

You can also query custom metric results from the JSON results. For example, to get the aggregated results as:


<CodeGroup labels={["Average", "Min", "Max"]} lineNumbers={[false]} showCopyButton={[true]} heightTogglers={[true]}>

```bash
jq '. | select(.type == "Point" and .metric == "coinflip duration") | .data.value' results.json | jq -s 'add/length'
```

```bash
jq '. | select(.type == "Point" and .metric == "coinflip duration") | .data.value' results.json | jq -s min
```

```bash
jq '. | select(.type == "Point" and .metric == "coinflip duration") | .data.value' results.json | jq -s max
```

</CodeGroup>


## Next steps

In this tutorial, you looked at granular output and filtered by built-in and custom tags.
Then you made a new script with groups.
Finally, you added a new metric for each group.
A next step would be to create a [Custom end-of-test summary](/results-output/end-of-test/custom-summary/) or to [stream the results to a database](/results-output/real-time/#service).

For ongoing operations, you can modularize your logic and configuration.
That's the subject of the [next step of this tutorial](/examples/tutorials/get-started-with-k6/reuse-and-re-run-tests/).
