#  Circuit Breaker

Another typical usecase is the implementation of circuit breaker patterns to avoid too many connections to a particular service, which could cause the service to misbehave for all users.

The goal here is to simulate a max number of concurrent connections to the productpage service of 1, and trip that circuit breaker with a load generator to verify its functionality. 

This is done with DestinationRules rather than VirtualServices (remember that DestinationRules group together services and apply policies for what happens to traffic for a given destination whereas VirtualServices define how you route traffic to a given destination).

Use the following template and the documentation for [ConnectionPools](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings). Set the Max Pending Requests and Max Requests per Connection to 1.

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  FILL_HERE
  subsets:
  - name: v1
    labels:
      version: v1
```

To validate it, deploy a `fortio` (a load generation tool) pod using the following command: 

```
kubectl apply -f solutions/fortio.yml
```

Then use the following commands to respectively do 20 requests serially, and then 20 requests with 3 concurrent requests.

```
export FORTIO_POD=$(kubectl get pods -lapp=fortio -o 'jsonpath={.items[0].metadata.name}')

kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 20 -loglevel Warning http://productpage:9080

kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 20 -loglevel Warning http://productpage:9080

```

You should see a 100% success in the first case and around a 33% success in the second case:

```
13:49:05 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 20 calls: http://productpage:9080
Starting at max qps with 3 thread(s) [gomax 2] for exactly 20 calls (6 per thread + 2)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
13:49:05 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 44.811527ms : 20 calls. qps=446.31
Aggregated Function Time : count 20 avg 0.0050252372 +/- 0.006042 min 0.000945184 max 0.024779832 sum 0.100504745
# range, mid point, percentile, count
>= 0.000945184 <= 0.001 , 0.000972592 , 10.00, 2
> 0.001 <= 0.002 , 0.0015 , 45.00, 7
> 0.002 <= 0.003 , 0.0025 , 55.00, 2
> 0.003 <= 0.004 , 0.0035 , 60.00, 1
> 0.004 <= 0.005 , 0.0045 , 65.00, 1
> 0.005 <= 0.006 , 0.0055 , 75.00, 2
> 0.006 <= 0.007 , 0.0065 , 80.00, 1
> 0.007 <= 0.008 , 0.0075 , 90.00, 2
> 0.018 <= 0.02 , 0.019 , 95.00, 1
> 0.02 <= 0.0247798 , 0.0223899 , 100.00, 1
# target 50% 0.0025
# target 75% 0.006
# target 90% 0.008
# target 99% 0.0238239
# target 99.9% 0.0246842
Sockets used: 15 (for perfect keepalive, would be 3)
Jitter: false
Code 200 : 6 (30.0 %)
Code 503 : 14 (70.0 %)
Response Header Sizes : count 20 avg 50.2 +/- 76.68 min 0 max 168 sum 1004
Response Body/Total Sizes : count 20 avg 723.8 +/- 737.5 min 241 max 1851 sum 14476
All done 20 calls (plus 0 warmup) 5.025 ms avg, 446.3 qps
```

The envoy proxy allows for some leeway so the percentage of successful request might vary a bit.

The solution for the destination rule can be found at [circuitbreaker-productpage-solution.yml](../solutions/circuitbreaker-productpage-solution.yml).

Reset the destination rules by applying the default destinations rules.

```
kubectl apply -f bookinfo/default-destinations.yml
```