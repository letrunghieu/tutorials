# Istio Circuit Breaker

## Install

```shell
helm install -n platform-staging istio-circuit-breaker helm/
helm upgrade -n platform-staging istio-circuit-breaker helm/ 
```

## Test

For the reference pods
```shell
kubectl -n platform-staging exec fortio-deploy-5d8fd4bb96-vcrps -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -t 0 http://httpbin:8000/delay/1````

For the main pod
```shell
kubectl -n platform-staging exec fortio-deploy-5d8fd4bb96-rprbv -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 http://httpbin:8000/delay/1
```

## Clean up