# Istio Circuit Breaker

## Prerequisites

* Istio
* Istio access log is turned on with JSON format
* One namespace with Istio auto-injection enabled
* (Optional) centralised logging, for example Loki/Grafana

## Install

```shell
helm install -n platform-staging istio-circuit-breaker helm/
```

Grafana query for Loki:

For downstream:

```text
{log_type="kubernetes-pods", namespace="platform-staging", app="fortio", container="istio-proxy"} |= `` | json | __error__=``
```

For upstream:

```text
{log_type="kubernetes-pods", namespace="platform-staging", app="httpbin", container="istio-proxy"} |= `` | json | label_format downstream_remote_ip="{{regexReplaceAll `(.*):.*`  .downstream_remote_address `$1`}}" | __error__=``
```
## Execute

### With circuit breaker disabled

Run `fortio` with maximum query per second (qps) and 10 connections, each request take 1 second to complete, for total 600 requests.
```shell
kubectl -n platform-staging exec fortio-0 -c fortio -- /usr/bin/fortio load -c 10 -qps 0 -n 600 http://httpbin:8000/delay/1
```

Expected result:

* There are ~960 requests in total ((60s / (1s/request)) * 16 connections)
* All requests are successful (100% response code 200)
* Upstream received 16 requests per second

### With circuit breaker enabled

Enabled the default destination rule:

```shell
 helm upgrade -n platform-staging --set circuitBreaker.enabled=true istio-circuit-breaker helm/
```
Run `fortio` with maximum query per second (qps) and 10 connections, each request take 1 second to complete, for total 600 requests.
```shell
kubectl -n platform-staging exec fortio-0 -c fortio -- /usr/bin/fortio load -c 10 -qps 0 -n 600 http://httpbin:8000/delay/1
```

Expected result:

* It takes less than 60s to complete all requests because the circuit breaker is triggered and stop sending requests to the upstream so that the trapped requests do not take 1 second to complete.
* There are many 503 responses because the circuit breaker is triggered and stop sending requests to the upstream.
* The number of requests reach the upstream if lower than the number of requests sent by the client.
* All request received by the upstream are successful (100% response code 200)

Now, retry with three cases:

* `-c 1 -n 60` (not tipping the circuit breaker)
* `-c 2 -n 120` (tipping the circuit breaker due to the number of connections = max + pending requests)
* `-c 3 -n 180` (tipping the circuit breaker due to the number of connections > max + pending requests)

#### How Istio circuit breaker works

In each pod, the application container call the `istio-proxy` instead of the upstream.
When a request reach the `istio-proxy`:

1. If there is no active connection:
   1. Create a new connection to the upstream.
2. If there is already some active connections:
   1. Check the number of requests handled by each connection.
      1. Send the request to the connection with the number of request less than `maxRequestsPerConnection`.
      2. Can queue to a connection if the total queued requests less than `http1MaxPendingRequests`.
3. If the request cannot be queued or handled by any open connection
   1. Check if the current number of reach the `maxConnections`:
      1. If yes, return 503.
      2. If no, create a new connection to the upstream (see 1.i.).

## Clean up